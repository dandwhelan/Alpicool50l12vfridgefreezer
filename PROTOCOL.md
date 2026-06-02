# Alpicool / Brass Monkey 12V Fridge — BLE Protocol

Reverse-engineered register/command map for Alpicool-style portable fridge freezers
(also sold as **Brass Monkey**, **BougeRV**, **SYDPOWER OEM**, etc.) controlled by the
**"CAR FRIDGE FREEZER"** Android/iOS app.

> ✅ **Verification status.** The frame format, checksum, command codes, the connection
> flow, **and the dual-zone 50 L status/settings byte map** below are now confirmed against a
> real `btsnoop_hci.log` capture of the official app driving an **A1-FFFF… 50 L dual-zone**
> unit (see §7). Current-temperature offsets in the dynamic tail are the only best-effort
> bytes left — confirm with **Diagnostics → Snapshot** if they look off on your unit.

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
  (The captured 50 L advertises as `A1-FFFF…`, GATT handles: notify `1236`→val 0x0003,
  write `1235`→val 0x0006, plus an extra `fff1` characteristic.)

### Connection / sync flow (verified)

1. Enable notifications on the `1236` CCCD.
2. **Send `QUERY` (0x01) immediately.** The fridge replies with a full status straight away —
   **no `BIND` (0x00) is sent or needed.** (Sending an unsolicited bind on connect is what
   made the dual-zone unit look "stuck pairing"; don't do it automatically.)
3. Poll `QUERY` every ~2 s.
4. **Chunk every write into ≤20-byte ATT packets**, even when a larger MTU is negotiated —
   the official app always does, and the fridge reassembles by the `FE FE`/length header. A
   single oversized write (e.g. the 31-byte settings frame) is silently dropped on a 20-byte
   link, so power/lock/unit changes appear to do nothing.

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
| `00` | **bind** | — | Optional "identify" handshake. **Not used in normal operation** — the captured app never sends it; only offer it as a manual fallback. |
| `01` | **query** | — | Request full status. App polls this ~every 2 s. Reply is `FE FE 21 01 …` (30-byte payload after the cmd byte). |
| `02` | **set** | 25-byte settings block | Write all general settings (power, lock, run-mode, battery-saver, unit). Echoed back as a `02` reply. |
| `04` | **reset** | — | Factory reset. ⚠️ destructive. |
| `05` | **setFridge** | `temp` (int8) | Set the **fridge** zone target. e.g. `FE FE 04 05 03 02 08` = +3°. |
| `06` | **setFreezer** | `temp` (int8) | Set the **freezer** zone target. e.g. `FE FE 04 06 EF 02 F5` = −17°. |

Temperatures are signed 8-bit (two's complement). **Setpoints (`05`/`06`, status idx 4 & 18)
are stored in the unit currently shown on the fridge** — switch to °F and `5` becomes `0x29`
(41 °F). The **current temperatures are reported in °C** regardless of the display unit, so
the app converts them for display when °F is selected.

`05` and `06` each take a **single temp byte** — *not* the full settings block. (Sending the
whole block with a `05`/`06` header is wrong and was a bug in this app.)

**Setting one option** (power, run-mode, battery protection, lock, unit) has no dedicated
command: take the most recent status, rebuild the 25-byte settings block (§5), flip the one
byte, and resend it as a `02`.

---

## 4. Status response (`01` query reply) — verified dual-zone 50 L

30-byte payload **after** the `01` command byte (frame `FE FE 21 01 …`, total 36 bytes,
fragmented over BLE as 20 + 16). The 25-byte settings block is interleaved with 5 dynamic
reading bytes (idx 14–17 and 26).

| Idx | Field | Type | Notes |
|:----|:------|:-----|:------|
| 0 | `locked` | bool | keypad lock |
| 1 | `powered` | bool | compressor on/off |
| 2 | `run_mode` | uint8 | `0` Max · `1` Eco |
| 3 | `batt_saver` | uint8 | `0` Low · `1` Mid · `2` High |
| 4 | **`fridge_target`** | int8 | fridge setpoint (set by `05`), in displayed unit |
| 5–11 | *settings* | mixed | limits / hysteresis / start-delay / calibration |
| 12 | `unit` | uint8 | `1` = °C · `2` = °F |
| 13 | *settings* | uint8 | (`00` observed) |
| **14–17** | *dynamic* | mixed | inserted readings (battery-ish / fridge current temp + tenths) |
| 16 | `fridge_current` | int8 | fridge current temp (°C) — best-effort offset |
| **18** | **`freezer_target`** | int8 | freezer setpoint (set by `06`), in displayed unit |
| 19–25 | *settings* | mixed | freezer-side limits / calibration |
| **26** | `freezer_current` | int8 | freezer current temp — best-effort offset |
| 27–29 | *settings* | mixed | tail (`00 03 00` observed) |

> Confirmed by replaying the capture: `parseStatus` of the real notification yields
> `fridge_target`, `freezer_target`, `unit` and the lock/power/run/batt flags exactly as the
> official app showed them. The `fridge_current`/`freezer_current` offsets (16 & 26) are the
> only soft spots — verify with **Diagnostics → Snapshot** if needed.

---

## 5. Set-settings block (`02`)

The 25-byte block is the status payload with the 5 dynamic bytes **removed**:

```
settings[0..13]  = status[0..13]
settings[14..21] = status[18..25]      // skip dynamic status[14..17]
settings[22..24] = status[27..29]      // skip dynamic status[26]
```

To change one option, rebuild this block from the latest status, flip its byte, and send:

```
FE FE 1C 02  <25-byte settings block, one byte changed>  CHK CHK     (31 bytes → write as 20 + 11)
```

This was verified end-to-end: reconstructing the block from a "power on" status and flipping
idx 1 reproduces the captured power-**off** `02` frame byte-for-byte.

---

## 6. Sources

The byte map and connection flow in this doc are **primarily derived from a first-party
`btsnoop_hci.log` capture** of the official *CAR FRIDGE FREEZER* app driving the physical
`A1-FFFF…` 50 L dual-zone unit, then cross-checked against:

- **klightspeed/BrassMonkeyFridgeMonitor** — Python, extracted from *CAR FRIDGE FREEZER* app **v2.0.0**.
- **johnelliott/alpicoold** — Go protocol package + HomeKit bridge, derived from Wireshark sniffing.

All three agree on the frame format and command codes. The capture additionally pinned down
the dual-zone setpoint/current layout and the 20-byte write-chunking behaviour.

---

## 7. Capture analysis (this repo)

The fixes in `index.html` were validated against the capture with a standalone replay
(`13/13` checks): frame builders match the wire bytes, the status parses at the correct
30-byte length, the settings reconstruction reproduces a captured `02` frame, and a settings
frame chunks into `[20, 11]`. The earlier `>= 31` length gate was an **off-by-one** (real
payload is 30) that dropped every status into a "waiting for bind" fallback — the root cause
of the unit appearing stuck in pairing.
