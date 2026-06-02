# Fossibot / Aferiy Power Station — BLE Protocol

The power-station half of this app speaks the **Fossibot / Aferiy / SYDPOWER** BLE protocol.
This spec is reimplemented faithfully from the sibling project
[**fossibot-bluetooth**](https://github.com/dandwhelan/fossibot-bluetooth) (its running
`index.html` + `PROTOCOL.md`), re-verified here against known-good CRCs.

> ⚠️ **The firmware does NOT validate writes.** An out-of-range value can permanently brick a
> unit. **Writing `0` to register 68 has bricked devices in the field** — this app hard-blocks
> it. Every control clamps to a safe whitelist.

---

## 1. Connection

| Role | UUID | Notes |
|:-----|:-----|:------|
| **Service** | `0000a002-0000-1000-8000-00805f9b34fb` | primary service |
| **Write** | `0000c304-0000-1000-8000-00805f9b34fb` | commands → station |
| **Notify** | `0000c305-0000-1000-8000-00805f9b34fb` | telemetry ← station |

Name prefixes: `POWER`, `AFERIY`, `FOSSIBOT`, `SYDPOWER`. All multi-byte values **big-endian**.

---

## 2. Frame format (Modbus-like)

```
[ 0x11 | func | regHi regLo | valHi valLo | CRChi CRClo ]
```

- `0x11` = fixed slave address.
- `func`: `0x03` read settings · `0x04` read status · `0x06` write single register · `0x07` WiFi config.
- **CRC = CRC-16/MODBUS** (poly `0xA001`, init `0xFFFF`), appended **high byte first** (non-standard byte order — required by the device).

```js
function crc16(b){ let t=0xFFFF; for(const x of b){ t^=x;
  for(let i=0;i<8;i++) t=(t&1)?((t>>1)^0xA001):(t>>1);} return t&0xFFFF; }
```

### Verified vectors

| Purpose | Bytes |
|:--------|:------|
| Poll **status** (read 80 regs fn 04) | `11 04 00 00 00 50 A6 F2` |
| Request **settings** (read 80 regs fn 03) | `11 03 00 00 00 50 66 47` |
| USB on (write reg 24 = 1) | `11 06 00 18 00 01 9D CA` |
| AC on (write reg 26 = 1) | `11 06 00 1A 00 01 5D 6B` |
| Discharge limit 10% (reg 66 = 100) | `11 06 00 42 00 64 A5 2A` |
| Charge limit 100% (reg 67 = 1000) | `11 06 00 43 03 E8 30 7A` |

---

## 3. Telemetry (read)

- Poll `11 04 00 00 00 50 A6 F2` to the write characteristic **every 2 s**; request settings
  `11 03 …` **once** on connect.
- Notifications arrive on `c305`. Dispatch on byte `[1]`: `0x04` = status (0x1104),
  `0x03` = settings (0x1103).
- Payload = **6 header bytes** then 16-bit big-endian registers.
  **`value = getUint16(6 + index*2)`**.

### Key status registers (0x1104)

| Reg | Field | Scaling |
|:----|:------|:--------|
| 6 | Total input W | raw (fallback reg3 + reg4) |
| 8 | Error code | 0 ok · 78 inverter fault · 79 temp/safety |
| 13 | AC charge rate | level 1–5 |
| 20 | Total output W | raw |
| 27 | LED mode | 0 off · 1 on · 2 flash · 3 SOS |
| 41 | Output flags | USB = bit 9 · DC = bit 10 · AC = bit 11 · LED = bit 12 |
| 42 | Protection mask | bits 13–14 (`& 0x6000`) = critical fault |
| 48 | Status flags | `0x8000` charging · `0x4000` standby |
| 54 | Battery full capacity | 0.1 Ah |
| 56 | **Main SoC** | **% = raw / 10** |
| 57 | Silent charging | 0/1 |
| 58 | Time to full | minutes |
| 59 | Time to empty | minutes |
| 66 | Discharge limit | % × 10 |
| 67 | Charge limit | % × 10 |

---

## 4. Control (write, fn 0x06) + safety whitelist

| Action | Reg | Allowed values |
|:-------|:----|:---------------|
| USB output | 24 | 0 / 1 |
| DC output | 25 | 0 / 1 |
| AC inverter | 26 | 0 / 1 |
| LED mode | 27 | 0 / 1 / 2 / 3 |
| AC charge rate | 13 | 1–5 |
| Max charge current (A) | 20 | 1–20 |
| Silent charging | 57 | 0 / 1 |
| Screen timeout | 59 | 0, 180, 300, 600, 1800 |
| AC standby (min) | 60 | 0, 480, 960, 1440 |
| DC standby (min) | 61 | 0, 480, 960, 1440 |
| USB standby | 62 | 0, 3, 5, 10, 30 |
| Discharge limit | 66 | 0–1000 (0.1 %) |
| Charge limit | 67 | 0–1000 (0.1 %) |
| **Idle shutdown (min)** | 68 | **5, 10, 30, 60, 480 — NEVER 0** |
| Power off now | 64 | 1 |

The app refuses any value outside these sets, and specifically blocks `reg 68 = 0`.

---

## 5. Source

Reimplemented from [dandwhelan/fossibot-bluetooth](https://github.com/dandwhelan/fossibot-bluetooth)
(`index.html`, `PROTOCOL.md`). CRC and command framing re-verified against the project's own
worked examples. See also [schauveau/sydpower-mqtt](https://github.com/schauveau/sydpower-mqtt)
for independent corroboration of the register map and the reg-68 brick hazard.
