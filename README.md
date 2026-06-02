# Campervan Control — Web Bluetooth

Offline, app-free, cloud-free control of a campervan's two Bluetooth systems from one phone-friendly
Progressive Web App:

- ❄️ **Alpicool / Brass Monkey** 12V fridge freezer
- 🔋 **Fossibot / Aferiy** portable power station

Built on the **Web Bluetooth API** — same architecture as
[**fossibot-bluetooth**](https://github.com/dandwhelan/fossibot-bluetooth), which this project
folds in so both devices live in one dashboard. No account, no internet; the app talks directly
to each device over BLE.

> The fridge and power station are **two completely separate Bluetooth systems**, so they each
> have their own **Connect** button and their own independent connection. Connect whichever
> you're near — or both at once.

---

## AI Assistant Integration
This project provides specific guidelines and docs for AI assistants to collaborate:
- [Claude](claude.md)
- [Gemini](gemini.md)
- [Codex](codex.md)

---

## Features

**Home** — both devices at a glance, each with its own connect/disconnect.

**❄️ Fridge** — current temp, target setpoint, battery %, voltage; set target temperature, power,
Max/Eco, battery protection, °C/°F, keypad lock.

**🔋 Power station** — battery SoC ring, input/output watts, time-to-full / time-to-empty; toggle
AC inverter, DC and USB outputs, LED modes, AC charge rate (1–5), charge & discharge SoC limits,
silent charging, standby timers — all behind a **safety whitelist** (out-of-range writes refused;
the device-bricking `reg 68 = 0` is hard-blocked).

**🛠️ Diagnostics** — combined activity log, per-device raw-command sender, the fridge
snapshot/change-recorder, and the live power-station register table.

📲 Installable · ⚡ offline · 🎨 four themes (Ocean, Midnight, Arctic, Terminal).

---

## Use it

### On your Pixel 9 Pro XL (recommended)
1. Open **`https://dandwhelan.github.io/Alpicool50l12vfridgefreezer/`** in **Chrome**
   *(enable GitHub Pages first — see below)*.
2. Turn on Bluetooth, stay in range.
3. Go to **Home**, tap **Connect** on the fridge and/or the power station (use "show all devices"
   if it isn't obviously named).
4. Optional: Chrome menu → **Add to Home screen** to install it.

### Run locally
Web Bluetooth needs a secure context (HTTPS **or** `localhost`):
```bash
python3 -m http.server 8000   # then browse to http://localhost:8000
```

### Enable GitHub Pages
*Settings → Pages → Deploy from a branch → `main` / root.* Plain static files, nothing to build.

> **Browser support:** Chrome/Edge on Android, Windows, macOS, Linux. **iOS** needs *Bluefy*.
> Desktop needs a Bluetooth adapter.

---

## How it works

Two protocols, two connections:

| Device | Service | Frame |
|:-------|:--------|:------|
| Fridge | `1234` (w `1235` / n `1236`) | `FE FE \| len \| payload \| sum16` |
| Power | `a002` (w `c304` / n `c305`) | `11 \| fn \| reg \| val \| CRC16` (Modbus) |

Full references: **[PROTOCOL.md](PROTOCOL.md)** (fridge) · **[POWER_PROTOCOL.md](POWER_PROTOCOL.md)** (power station).
Both protocols' framing/CRCs are **verified** against known-good packets. The fridge per-byte
status map is best-effort pending validation on the physical 50 L (use Diagnostics → Snapshot).

---

## Project layout

| File | Purpose |
|:-----|:--------|
| `index.html` | the whole app (Home / Fridge / Power / Diagnostics, inline CSS+JS) |
| `manifest.json` · `service-worker.js` · `icon.svg` | PWA install + offline |
| `PROTOCOL.md` | fridge (Alpicool) protocol |
| `POWER_PROTOCOL.md` | power station (Fossibot) protocol |
| `task.md` | roadmap / status |

---

## Status

🟢 **Fridge protocol verified against a real `btsnoop_hci.log` capture** of the official app
driving the A1-FFFF… 50 L dual-zone unit — connection/sync, command codes and the dual-zone
byte map are confirmed (current-temp offsets remain best-effort; see `PROTOCOL.md`). 🟡 The
power-station protocol is CRC-verified but not yet validated on hardware; its writes are gated
by a safety whitelist — treat untested controls with care. See `task.md`.

## Credits

- Fridge: [klightspeed/BrassMonkeyFridgeMonitor](https://github.com/klightspeed/BrassMonkeyFridgeMonitor), [johnelliott/alpicoold](https://github.com/johnelliott/alpicoold)
- Power: [dandwhelan/fossibot-bluetooth](https://github.com/dandwhelan/fossibot-bluetooth), [schauveau/sydpower-mqtt](https://github.com/schauveau/sydpower-mqtt)

## ⚠️ Safety

Unofficial software. The fridge stores food/medicine — verify temperatures against its own
display. The power station's firmware does **not** validate writes; bad values can brick it
(notably `reg 68 = 0`, which this app blocks). Use at your own risk.

## License

MIT
