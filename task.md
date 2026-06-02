# Task / roadmap

## ✅ Phase 1 — foundation (done)
- [x] Research prior art (BrassMonkeyFridgeMonitor, alpicoold)
- [x] Verify frame format + checksum against known-good packets
- [x] Web Bluetooth PWA scaffold (matches fossibot-bluetooth architecture)
- [x] Connect / disconnect, name-prefix + "all devices" chooser
- [x] GATT discovery dump (confirm `1234/1235/1236` vs `ffe0/ffe1`)
- [x] Query polling + status decode (best-effort)
- [x] Controls: target temp, power, Max/Eco, battery protection, lock, unit
- [x] Diagnostics: raw log, decoded byte table, snapshot/change-recorder, raw-command sender
- [x] PWA install + offline (manifest, service worker, icon)
- [x] PROTOCOL.md + README

## 🔲 Phase 2 — validate on the real 50 L (needs the physical fridge)
- [ ] Confirm GATT profile is `1234/1235/1236` (capture from Pixel + report in GATT discovery)
- [ ] Confirm status byte map: current temp, battery %, voltage offsets
- [ ] Confirm °C/°F flag byte index (currently assumed idx 11 — see `unitByteIndex()`)
- [ ] Confirm settings-block layout for `02` set (power/eco/lock all verified working)
- [ ] Determine single- vs dual-zone for this 50 L model
- [ ] Capture a btsnoop HCI log of the official app for any commands we're missing

### How to capture from the Pixel 9 (for phase 2)
1. Settings → About phone → tap **Build number** 7× to enable Developer options.
2. Developer options → enable **Bluetooth HCI snoop log** (set to *Enabled/Filtered*).
3. Use the official **CAR FRIDGE FREEZER** app: connect, change temp, toggle power, etc.
4. Developer options → **Generate bug report** (or pull `btsnoop_hci.log`), open in **Wireshark**.
5. Filter `btatt`, match the `1235` writes / `1236` notifications against actions.

## 🔲 Phase 3 — polish & integrate
- [ ] Temperature history graph
- [ ] Shared theming/components with fossibot-bluetooth
- [ ] Combined "campervan" launcher linking both PWAs (or a unified multi-device dashboard)
- [ ] Replace SVG icon with a proper PNG icon set
