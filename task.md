# Task / roadmap

## ✅ Phase 1 — fridge foundation (done)
- [x] Research prior art; verify fridge frame + checksum
- [x] Web Bluetooth PWA scaffold
- [x] Fridge connect/disconnect, query polling, status decode (best-effort)
- [x] Fridge controls + diagnostics (raw log, byte table, snapshot recorder)

## ✅ Phase 2 — unified app (done)
- [x] Reverse-engineer Fossibot protocol from fossibot-bluetooth (CRC + registers verified)
- [x] Rebuild as **Campervan Control**: Home / Fridge / Power / Diagnostics
- [x] **Separate Connect buttons** + independent BLE connections per device
- [x] Power: SoC/watts/time dashboard; AC/DC/USB/LED/charge-rate/limits/silent/timers controls
- [x] Power safety whitelist + hard-block on reg-68=0 (brick guard)
- [x] Power register table + raw command sender in diagnostics
- [x] Docs: PROTOCOL.md (fridge) + POWER_PROTOCOL.md (power)

## 🟡 Phase 3 — validate on real hardware
**Fridge — verified from a first-party `btsnoop_hci.log` capture of the A1-FFFF… 50 L:**
- [x] Confirmed GATT profile `1234` (notify `1236`/val 0x0003, write `1235`/val 0x0006, + `fff1`)
- [x] Confirmed it's **dual-zone**: fridge target via `05`, freezer target via `06`
- [x] Confirmed `02` settings-block layout (25 bytes; dynamic readings interleaved) + °C/°F idx 12
- [x] Fixed connect/sync: 30-byte status gate (was off-by-one `31`), no auto-bind, 20-byte write chunking
- [x] **Validated on the physical 50 L** — connect, fridge/freezer temp set, power toggle all confirmed working
- [x] Confirmed current-temp offsets on hardware: fridge `idx16/17`, freezer `idx26/27` (whole + tenths, °C)

**Power (needs the Fossibot):**
- [ ] Confirm connect + live telemetry (SoC, watts, time)
- [ ] Confirm each output toggle (AC/DC/USB/LED) round-trips
- [ ] Sanity-check charge/discharge limit writes before relying on them

### Capturing from the Pixel 9 (for verification)
1. Settings → About phone → tap **Build number** ×7 → Developer options.
2. Enable **Bluetooth HCI snoop log**.
3. Use the official apps (CAR FRIDGE FREEZER / BrightEMS) to perform actions.
4. **Generate bug report** / pull `btsnoop_hci.log`, open in **Wireshark**, filter `btatt`.

## 🔲 Phase 4 — polish
- [ ] Temperature & SoC history graphs
- [ ] Combined "scenes" (e.g. low-power night mode across both devices)
- [ ] Proper PNG icon set
- [ ] Fold improvements back toward fossibot-bluetooth where shareable
