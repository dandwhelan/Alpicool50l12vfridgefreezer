# Alpicool / Brass Monkey 12V Fridge — BLE Protocol

Register/command map for Alpicool-style portable fridge freezers (also sold as **Brass Monkey**,
**BougeRV**, etc.) controlled by the **"CAR FRIDGE FREEZER"** app.

**Verification status: VERIFIED.** Derived by decompiling the Hermes bytecode (v96) in
`CAR+FRIDGE+FREEZER_2.3.7` (`assets/index.android.bundle`, via `hermes-dec`) and cross-checked
against a real HCI snoop capture (`btsnoop_hci.log.last`) from a dual-zone unit (`model = 3`).
Every frame in the capture passes the checksum and decodes cleanly with the layout below.
This supersedes the earlier byte map, which had the tail fields in the wrong place.

---

## 1. Connection

| Role | UUID | Notes |
|:-----|:-----|:------|
| **Service** | `00001234-0000-1000-8000-00805f9b34fb` | |
| **Write** | `00001235-0000-1000-8000-00805f9b34fb` | ATT **Write Request** (with response) |
| **Notify** | `00001236-0000-1000-8000-00805f9b34fb` | enable CCCD (`01 00`) first |

- No pairing/PIN. Only one client at a time.
- Name prefixes the app accepts (`checkBleName`): `MF-`, `A1-`, `WT-0001`, `AK1-`, `AK2-`, `AK3-`.
  Names starting `KGS-`, `KGD-`, `W-`, `WH-` are treated as needing "activation".
- **Writes are split into 20-byte chunks** (default 23-byte MTU). The app writes bytes 0–19, then the
  rest, as separate Write Requests. The fridge reassembles using the `LEN` byte. Notifications
  longer than 20 bytes likewise arrive in two parts — buffer until `len == LEN + 3`.
- Connect sequence seen in capture: enable notifications → send `query` → poll `query`
  (app uses a `setInterval` timer). `bind` (`FE FE 03 00 01 FF`) is only used by the app's
  pairing flow; it is **not required** to read status or send commands.

---

## 2. Frame format (both directions)

```
FE FE | LEN | CMD | params... | SUM_hi SUM_lo
```

- `LEN` = `1 (CMD) + params + 2` = total frame length − 3.
- `SUM` = 16-bit big-endian sum of every byte before it (`FE FE LEN CMD params…`).
- Receiver checks: header `FE FE`, `LEN == frame.length − 3`, checksum.

Signed values (`int8`) are two's complement: `0xEC = -20`, `0x80 = -128` = "no sensor" (`EMPTY_DEGREE`).

---

## 3. Commands (`CMD`)

| CMD | Name | Params | Response |
|:----|:-----|:-------|:---------|
| `00` | bind | – | `00 <flag…>` |
| `01` | **query** | – | full status (§4) |
| `02` | **set others** | full settings block (§5) | full status |
| `04` | reset (factory) ⚠ | – | |
| `05` | **set left target** | `temp` int8 | echo `05 temp` |
| `06` | **set right target** | `temp` int8 | full status |
| `07` | set hot target | `temp` int8 | |
| `08` | check bind | – | bind info + 14-byte MAC |
| `09` | set bind | – | |

Examples (captured):

| Action | Bytes |
|:-------|:------|
| Query | `FE FE 03 01 02 00` |
| Left target → 3 °C | `FE FE 04 05 03 02 08` |
| Right target → −18 °C | `FE FE 04 06 EE 02 F4` |

---

## 4. Status response (`CMD 01`) — payload index after the CMD byte

Full frame is 33 bytes on dual-zone units (`LEN = 0x21`): 30 payload bytes (idx 0–29). Idx 30–33
(`warn`, `sleep`, `partition`, `disinfect`) exist only on newer models when the payload is > 36 bytes.
"Left" = zone 1 (fridge / main), "Right" = zone 2 (freezer) on dual-zone units.

| Idx | Field | Type | Meaning / captured value |
|:----|:------|:-----|:-------------------------|
| 0 | `locked` | u8 | keypad lock 0/1 |
| 1 | `poweredOn` | u8 | 0 off / 1 on |
| 2 | `runMode` | u8 | 0 = Max, 1 = Eco |
| 3 | `batSaver` | u8 | 0 Low / 1 Mid / 2 High |
| 4 | `leftTarget` | i8 | zone-1 setpoint |
| 5 | `tempMax` | i8 | zone-1 max selectable (8) |
| 6 | `tempMin` | i8 | zone-1 min selectable (0) |
| 7 | `leftRetDiff` | i8 | hysteresis / return difference (2 °C, 4 °F) |
| 8 | `startDelay` | u8 | compressor start delay, minutes |
| 9 | `unit` | u8 | **0 = °C, 1 = °F** |
| 10 | `leftTCHot` | i8 | temp-compensation, hot ambient |
| 11 | `leftTCMid` | i8 | temp-compensation, mid |
| 12 | `leftTCCold` | i8 | temp-compensation, cold (1) |
| 13 | `leftTCHalt` | i8 | temp-compensation, halt |
| 14 | `leftCurrent` | i8 | **zone-1 current temperature** |
| 15 | `batPercent` | u8 | battery-input % (100) |
| 16 | `batVolInt` | u8 | supply volts, integer part (13) |
| 17 | `batVolDec` | u8 | supply volts, tenths → `13.1 V` |
| 18 | `rightTarget` | i8 | zone-2 setpoint (−16) |
| 19 | `rightMax` | i8 | zone-2 max selectable (−12) |
| 20 | `rightMin` | i8 | zone-2 min selectable (−20) |
| 21 | `rightRetDiff` | i8 | zone-2 hysteresis |
| 22–25 | `rightTCHot/Mid/Cold/Halt` | i8 | zone-2 compensation |
| 26 | `rightCurrent` | i8 | **zone-2 current temperature** (−128 = no sensor) |
| 27 | `runningStatus` | u8 | compressor/run state |
| 28 | `model` | i8 | model id (3 on the captured unit) |
| 29 | `hotTarget` | i8 | third-zone (hot) target, models with heating |
| 30–33 | `warn`, `sleep`, `partition`, `disinfect` | u8 | newer models only |

Single-zone units send < 19 payload bytes; the app then sets `rightCurrent = -128`.

Decode:
```js
const s8 = b => (b & 0x80) ? b - 256 : b;
const volts = p[16] + p[17] / 10;
```

### Units
`unit = 1` switches every temperature field to °F on the wire (capture: target 5 °C → `41`,
right −15 °C → `5`). Setpoints you send must be in the currently selected unit.

### Per-model limits (from app `ParamLimit`, [°C, °F])
Each entry is `[°C, °F]`; the app picks by `unit`.

| Field | °C | °F |
|:------|:---|:---|
| Absolute temp range (`tempMin`/`tempMax`) | −40 … 40 | −40 … 104 |
| Left target min bound | −10 | 14 |
| Right target min / max bound (fridge-type zone) | 0 / 8 | 32 / 46 |
| Hysteresis (`RetDiff`) | 1 … 10 | 2 … 18 |
| Start delay | 0 … 10 min | |
| Temp-compensation Hot/Mid/Cold | −10 … 10 | −18 … 18 |
| Temp-compensation Halt | −10 … 0 | −18 … 0 |
| Hot target | 25 … 50 | 77 … 122 |

Always clamp to the `tempMin`/`tempMax` (idx 5,6) and `rightMin`/`rightMax` (idx 19,20) the fridge
reports rather than the generic table.

---

## 5. Set-others block (`CMD 02`)

The app has no per-option command. To change lock/power/mode/battery/unit/hysteresis/delay/
compensation it sends **all settings** as one `02` frame built from the last status.

```
02
 [0] locked  [1] poweredOn  [2] runMode  [3] batSaver
 [4] leftTarget  [5] tempMax  [6] tempMin  [7] leftRetDiff
 [8] startDelay  [9] unit
 [10] leftTCHot  [11] leftTCMid  [12] leftTCCold  [13] leftTCHalt
 -- dual-zone / model != 1 only --
 [14] rightTarget [15] rightMax [16] rightMin [17] rightRetDiff
 [18] rightTCHot  [19] rightTCMid [20] rightTCCold [21] rightTCHalt
 [22] hotTarget   [23] model
 -- tail --
 model < 9 & model != 1 :  00
 model ≥ 9              :  runningStatus, sleep, disinfect, 00,00,00,00,00,00
```
i.e. the settings block is idx 0–13 of the status, then idx 18–25, `hotTarget`, `model`, then the tail.
Signed fields are sent as unsigned bytes (`v < 0 ? 256 + v : v`).

Captured example (power on → off, model 3), sent as two 20-byte writes:
```
FE FE 1C 02 00 00 01 02 04 08 00 02 00 00 00 00 01 00 F0 F4 | EC 02 00 00 01 00 00 03 00 05 02
             lk pw rm bs tg mx mn rd sd un tH tM tC tHl  rT  rMx  rMn rRd tH tM tC tHl hot mdl 0  CHK
```

### How-to for common controls
| Want | Do |
|:-----|:---|
| Power on/off | `02` with `poweredOn` byte flipped |
| Eco / Max | `02` with `runMode` (0 Max, 1 Eco) |
| Battery protection | `02` with `batSaver` (0/1/2) |
| Lock keypad | `02` with `locked` |
| °C ↔ °F | `02` with `unit` **and** all temp fields converted (the app converts them itself) |
| Zone-1 temp | `05` (or `02`) |
| Zone-2 temp | `06` |
| Hysteresis / start delay / compensation | `02` |

---

## 6. Corrections vs. the earlier draft

- Current temps are **idx 14 (zone 1)** and **idx 26 (zone 2)**, not "last 4 bytes".
- Battery/voltage are **idx 15–17**, not the tail.
- `unit` is idx 9, `startDelay` idx 8, `leftRetDiff` idx 7, TC bytes idx 10–13.
- Added commands `07` (hot target), `08` (check-bind), `09` (set-bind).
- Chunking to 20-byte writes is required.
- `bind` is not needed to control the fridge.

## 7. Sources / method

- Decompiled `assets/index.android.bundle` (Hermes v96) with `hermes-dec`; functions
  `calcChecksum`, `makeCommand`, `encodeSetOthers`, `fetchData`, `decodeResponse`, `ParamLimit`.
- `btsnoop_hci.log.last`: reassembled ATT writes/notifications; all checksums valid; each
  documented field was seen changing in response to the matching app action
  (power, run mode, battery saver, left/right targets, °C↔°F).
- Note: `btsnoop_hci.log` (first file) is a **different device** (Fossibot power station, `0x11 0x06…`
  Modbus-style frames) — see `POWER_PROTOCOL.md`.
- Not yet observed on the wire: `locked`, `startDelay`, TC/hysteresis edits, `hotTarget`, `reset`,
  and `warn/sleep/partition/disinfect` (layout comes from the app code only).

## 8. Notes from on-hardware validation (merged from earlier work)

An earlier pass validated this protocol on the physical `A1-FFFF…` 50 L dual-zone unit against a
first-party `btsnoop_hci.log` capture, and cross-checked it against
**klightspeed/BrassMonkeyFridgeMonitor** (app v2.0.0) and **johnelliott/alpicoold**.

- The status payload is **30 bytes after the cmd byte**, so parsers must gate on `>= 30`. A `>= 31`
  gate (an old off-by-one) sends every status to a "waiting for bind" fallback and the unit looks
  stuck pairing.
- Do **not** auto-send `bind` on connect; the official app sends `query` immediately.
- Chunk every write to ≤ 20 bytes; a single oversized write is silently dropped.
- Extra GATT characteristic seen on the unit: `fff1` (unused by the app).

**Superseded by the decompiled app (§4):** that pass inferred the current-temperature bytes by
elimination and concluded idx 16/17 "read ~4.9° high" and idx 15 was tenths. The app's
`fetchData()` shows idx 15 is **battery %** and idx 16/17 is **supply voltage** (12.9 V — the
"4.9° high" reading), while zone-1 current is **idx 14 (whole degrees, no tenths)** and zone-2
current is **idx 26**. Current temps are sent **in the display unit** (°F mode capture: idx 14 = 61
for 16 °C, idx 26 = 55 for 13 °C), not always °C.
