# Alpicool / Brass Monkey 12V Fridge — BLE Protocol

Reverse-engineered register/command map for Alpicool-style portable fridge freezers
(also sold as **Brass Monkey**, **BougeRV**, **SYDPOWER OEM**, etc.) controlled by the
**"CAR FRIDGE FREEZER"** Android/iOS app.

> ⚠️ **Verification status.** The *frame format and checksum* below are confirmed against
> known-good packets from the official app. The *per-byte status map* is corroborated across
> two independent projects but **may differ on the 50 L unit this repo targets**. Use the
> in-app **Diagnostics → Snapshot** tool to confirm byte offsets on your own hardware and
> update the tables here as findings land.

---

## 1. Connection

| Role | UUID | Notes |
|:-----|:-----|:------|
| **Service** | `00001234-0000-1000-8000-00805f9b34fb` | primary service |
| **Write** | `00001235-0000-1000-8000-00805f9b34fb` | commands → fridge (`write` / `write-without-response`) |
| **Notify** | `00001236-0000-1000-8000-00805f9b34fb` | status ← fridge (`notify`) |

- **No authentication / pairing / PIN** is required — the fridge accepts commands from any
  connected client.
- Connecting **locks out other BLE clients** until you disconnect (only one app at a time).
- Some models instead expose the generic `ffe0` / `ffe1` HM-10 profile. The app's
  **Diagnostics → GATT discovery** lists every service so you can confirm which profile your
  unit uses.
- Advertised name prefixes seen in the wild: `A1-`, `AK1-`, `AK2-`, `AK3-`, `WT-`, `K25`, `BC…`.

---

## 2. Frame format

Every packet — both directions — uses the same envelope:

| Header | Length | Payload | Checksum |
|:-------|:-------|:--------|:---------|
| `FE FE` | `LEN` | `CMD` + params | `SUM_hi` `SUM_lo` |

- **`LEN`** = `payload length + 2` (the `+2` counts the two checksum bytes).
- **`CMD`** = `payload[0]`, the command/response code.
- **Checksum** = 16-bit big-endian **sum of every byte before the checksum** (`FE` … last payload byte), masked to `0xFFFF`.

```js
function buildFrame(payload) {              // payload = [CMD, ...params]
  const len  = payload.length + 2;
  const body = [0xFE, 0xFE, len, ...payload];
  let sum = 0; for (const b of body) sum += b;
  sum &= 0xFFFF;
  return Uint8Array.of(...body, (sum >> 8) & 0xFF, sum & 0xFF);
}
```

### Verified examples

| Action | Bytes | Checksum check |
|:-------|:------|:---------------|
| Bind | `FE FE 03 00 01 FF` | `FE+FE+03+00 = 0x01FF` ✔ |
| Query | `FE FE 03 01 02 00` | `FE+FE+03+01 = 0x0200` ✔ |
| Set −20°C | `FE FE 04 05 EC 02 F1` | `FE+FE+04+05+EC = 0x02F1` ✔ |

---

## 3. Commands (`payload[0]`)

| Code | Name | Params | Purpose |
|:-----|:-----|:-------|:--------|
| `00` | **bind** | — | Confirm/pair fridge (press fridge **SET** button first). |
| `01` | **query** | — | Request full status. App polls this ~every 2 s. |
| `02` | **set** | full settings block | Write all general settings (echoes the settings half of the status). |
| `04` | **reset** | — | Factory reset. ⚠️ destructive. |
| `05` | **setLeft** | `temp` (int8) | Set unit-1 / main target temperature. |
| `06` | **setRight** | `temp` (int8) | Set unit-2 target (dual-zone units only). |

Temperatures are signed 8-bit (two's complement), in the unit currently selected on the
fridge (°C or °F). `0xEC = -20`.

**Setting one option** (power, eco, battery protection, lock, unit): the app does **not** have
a dedicated command. It takes the most recent status, replaces the relevant byte in the
*settings* region, and sends the whole block back as a `02` command. This app does the same
via the cached settings buffer.

---

## 4. Status response (`01` query reply)

Payload below is **after** the `01` command byte. Single-zone units return ~18 bytes;
dual-zone extends further. Dynamic readings (current temp, battery) sit in the **last 4 bytes**.

| Idx | Field | Type | Notes |
|:----|:------|:-----|:------|
| 0 | `locked` | bool | keypad lock |
| 1 | `powered` | bool | compressor on/off |
| 2 | `run_mode` | uint8 | `0` Max · `1` Eco |
| 3 | `batt_saver` | uint8 | `0` Low · `1` Mid · `2` High |
| 4 | `target` | int8 | setpoint (unit-1) |
| 5 | `max_temp` | int8 | max selectable |
| 6 | `min_temp` | int8 | min selectable |
| 7–13 | *settings* | mixed | hysteresis, start-delay, °C/°F flag, calibration offsets — **layout varies, verify** |
| n−4 | `current` | int8 | current temperature |
| n−3 | `batt_pct` | uint8 | battery % |
| n−2 | `volt_int` | uint8 | battery volts, whole part |
| n−1 | `volt_frac` | uint8 | battery volts, tenths → `volt_int + volt_frac/10` |

> The °C/°F flag and calibration bytes (idx 7–13) are the least certain. Confirm with the
> Snapshot tool: take a snapshot, toggle the setting in the official app, see which byte
> changes.

---

## 5. Set-settings block (`02`)

Mirrors the status layout but **omits the dynamic tail** (current temp + battery). i.e. the
settings block is `status_payload[0 .. n-5]`. To change one option, flip its byte and resend:

```
FE FE LEN  02  <settings block with one byte changed>  CHK CHK
```

---

## 6. Sources

This map was assembled from prior reverse-engineering work, then re-verified:

- **klightspeed/BrassMonkeyFridgeMonitor** — Python, extracted from *CAR FRIDGE FREEZER* app **v2.0.0** (the same app this repo targets). Note: app v2.2.9+ ships Hermes bytecode, so v2.0.0 was the last cleanly-extractable build.
- **johnelliott/alpicoold** — Go protocol package + HomeKit bridge, derived from Wireshark sniffing of the app.

Both agree on the frame format, command codes and the position of the dynamic readings.
