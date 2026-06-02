# Alpicool 50 L 12V Fridge Freezer — Web Bluetooth control

Offline, app-free, cloud-free control of an **Alpicool / Brass Monkey** 12V portable fridge
freezer straight from your phone's browser — a Progressive Web App built on the
**Web Bluetooth API**.

Sibling project to [**fossibot-bluetooth**](https://github.com/dandwhelan/fossibot-bluetooth)
and built the same way (single-file PWA, GitHub Pages, no backend) so the two can share
patterns and eventually a combined "campervan control" dashboard.

> Replaces the OEM **"CAR FRIDGE FREEZER"** app. No account, no internet, talks directly to
> the fridge over Bluetooth Low Energy.

---

## Features

- 🌡️ Read **current temperature**, target setpoint, battery %, and supply voltage
- 🎛️ Set **target temperature**, toggle **power**, switch **Max/Eco**, set **battery protection** & **keypad lock**
- 🔋 Live battery ring + on/off status
- 🛠️ **Diagnostics** view: raw packet log, GATT discovery, decoded byte table, raw-command sender, and a **Snapshot/change-recorder** to reverse-engineer remaining bytes on *your* exact unit
- 🎨 Four themes (Ocean, Midnight, Arctic, Terminal), 📲 installable as a standalone app, ⚡ works offline

---

## Use it

### On your Pixel 9 Pro XL (recommended)
1. Open **`https://dandwhelan.github.io/Alpicool50l12vfridgefreezer/`** in **Chrome**.
   *(Enable GitHub Pages first — see below. Until then, see "Run locally".)*
2. Turn on Bluetooth, stay within range of the fridge.
3. Tap **🔌**, pick your fridge from the chooser (if it isn't named obviously, choose **show all devices**).
4. Optional: Chrome menu → **Add to Home screen** to install it as an app.

### Run locally (any machine with Chrome/Edge)
Web Bluetooth needs a *secure context* (HTTPS **or** `localhost`):
```bash
python3 -m http.server 8000
# then browse to http://localhost:8000
```

### Enable GitHub Pages
Repo **Settings → Pages → Build and deployment → Source: Deploy from a branch →
`main` / root**. The PWA is plain static files, nothing to build.

> **Browser support:** Chrome/Edge on Android, Windows, macOS, Linux. **iOS** needs the
> *Bluefy* browser (Safari has no Web Bluetooth). Desktop needs a Bluetooth adapter.

---

## How it works

The fridge speaks a simple framed protocol over GATT service `1234`
(write `1235`, notify `1236`):

```
FE FE | LEN | CMD + params | CHK_hi CHK_lo      CHK = 16-bit big-endian sum
```

Full register/command map: **[PROTOCOL.md](PROTOCOL.md)**.

The frame format and checksum are **verified** against the official app. The per-byte status
map is best-effort and may vary on this 50 L unit — the Diagnostics **Snapshot** tool is there
to confirm it on real hardware (snapshot → change one setting → watch which byte highlights).

---

## Project layout

| File | Purpose |
|:-----|:--------|
| `index.html` | The entire app (inline CSS + JS) |
| `manifest.json` / `service-worker.js` / `icon.svg` | PWA install + offline |
| `PROTOCOL.md` | Reverse-engineered protocol reference |
| `task.md` | Roadmap / status checklist |

---

## Status

🟡 **Phase 1 — first working build.** Connect, read status, set temperature, and full
diagnostics tooling are implemented from documented prior art. **Not yet validated against the
physical 50 L fridge** — see `task.md` for the verification checklist.

## Credits & references

- [klightspeed/BrassMonkeyFridgeMonitor](https://github.com/klightspeed/BrassMonkeyFridgeMonitor) — protocol extracted from *CAR FRIDGE FREEZER* v2.0.0
- [johnelliott/alpicoold](https://github.com/johnelliott/alpicoold) — Go protocol + HomeKit bridge

## ⚠️ Safety

Unofficial software for a device that stores food/medicine. Verify temperatures against the
fridge's own display before relying on it. The `reset` (`04`) command factory-resets the unit.
Use at your own risk.

## License

MIT
