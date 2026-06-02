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

## 🔲 Phase 3 — validate on real hardware
**Fridge (needs the physical 50 L):**
- [ ] Confirm GATT profile `1234/1235/1236` (vs `ffe0/ffe1`)
- [ ] Confirm status byte map (current temp / battery / voltage offsets, °C/°F flag idx)
- [ ] Confirm `02` settings-block layout; single- vs dual-zone

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
