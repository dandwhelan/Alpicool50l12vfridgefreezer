# Campervan Control — Web Bluetooth

Offline, app-free, cloud-free control of a campervan's two Bluetooth systems from one phone-friendly
Progressive Web App:

- ❄️ **Alpicool / Brass Monkey** 12V fridge freezer *(+ many rebrands — see [COMPATIBILITY.md](COMPATIBILITY.md))*
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
4. Install it: tap the **⬇️ Install** button (in the header / Home screen) when it
   appears, or use Chrome menu → **Add to Home screen**. Once installed it launches
   full-screen and runs entirely offline.

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

## Compatible fridges

This app speaks the **Alpicool "CAR FRIDGE FREEZER" BLE protocol** (GATT service `1234`,
`FE FE … sum16` frames). The same hardware is rebranded widely, so it works with far more than
just Alpicool. Advertised BLE name usually starts with **`A1-`, `AK1-`, `AK2-`, `AK3-`, `WT-`,
`K25`, `BC…`**.

> **The one reliable test:** does it pair with the official *CAR FRIDGE FREEZER* app **and** show
> service **`1234`** (write `1235` / notify `1236`) under **Diagnostics → GATT discovery**? If yes,
> this app drives it. A different app, or generic `ffe0/ffe1` (HM‑10), means a different lineage.

**✅ Confirmed (same app + protocol)**

| Brand | Notes |
|:------|:------|
| **Alpicool** | The OEM/original. The 50 L dual-zone this repo was built against is an Alpicool. |
| **Brass Monkey** | AU/NZ rebrand (Jaycar, Bunnings, Road Tech Marine). Same hardware + app. |
| **BougeRV** (CR-series) | Named in the official app listing. ⚠️ Some BougeRV models use a different (Wancool/SECOP) stack — verify with the test. |

**🟡 Likely same-platform rebrands (verify with the test above)**

Setpower · JoyTutus · Bodega · Euhomy · Vevor · Costway · Aspenora · Ausranvik · AstroAI ·
Kalamera · Domende — and other budget Amazon car-fridge brands. Commonly described as
Alpicool-made or sharing the platform, but the GATT `1234` check is the only proof.

**❌ Different ecosystem (won't work)**

ICECO (and ICECO-built Setpower) · ARB · Dometic · National Luna · Engel · EcoFlow Glacier ·
Anker — all use their own controllers/apps/protocols.

> Full details, sources, and how to report a new one: **[COMPATIBILITY.md](COMPATIBILITY.md)**.
> On the power side, the app targets **Fossibot / Aferiy / SYDPOWER** stations (name prefixes
> `POWER`, `AFERIY`, `FOSSIBOT`, `SYDPOWER`).

---

## Project layout

| File | Purpose |
|:-----|:--------|
| `index.html` | the whole app (Home / Fridge / Power / Diagnostics, inline CSS+JS) |
| `manifest.json` · `service-worker.js` · `icon.svg` · `icon-*.png` | PWA install (incl. maskable icon) + offline |
| `PROTOCOL.md` | fridge (Alpicool) protocol |
| `POWER_PROTOCOL.md` | power station (Fossibot) protocol |
| `COMPATIBILITY.md` | which fridge brands/rebrands work with this app |
| `task.md` | roadmap / status |

---

## Status

🟢 **Fridge validated on the physical 50 L dual-zone unit** — connect/sync, the dual-zone byte
map, fridge/freezer temperature set and the power toggle are all confirmed working against a
real session of the official app (see `PROTOCOL.md`). 🟡 The power-station protocol is
CRC-verified but not yet validated on hardware; its writes are gated by a safety whitelist —
treat untested controls with care. See `task.md`.

## Credits

- Fridge: [klightspeed/BrassMonkeyFridgeMonitor](https://github.com/klightspeed/BrassMonkeyFridgeMonitor), [johnelliott/alpicoold](https://github.com/johnelliott/alpicoold)
- Power: [dandwhelan/fossibot-bluetooth](https://github.com/dandwhelan/fossibot-bluetooth), [schauveau/sydpower-mqtt](https://github.com/schauveau/sydpower-mqtt)

## ⚠️ Safety

Unofficial software. The fridge stores food/medicine — verify temperatures against its own
display. The power station's firmware does **not** validate writes; bad values can brick it
(notably `reg 68 = 0`, which this app blocks). Use at your own risk.

## License

MIT
