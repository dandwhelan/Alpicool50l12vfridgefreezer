# claude.md — guide for AI assistants working on this repo

Campervan Control is a single-file **Web Bluetooth PWA** that talks to two unrelated BLE
devices from one phone: an **Alpicool / Brass Monkey 50 L dual-zone fridge** and a
**Fossibot power station**. Everything lives in `index.html` (inline CSS + JS, two IIFE
modules: `Fridge` and `Power`). There is no build step — it's static files served over
HTTPS/`localhost`.

## Layout

| File | What |
|:-----|:-----|
| `index.html` | the whole app — Home / Fridge / Fridge-settings / Power / Diagnostics |
| `PROTOCOL.md` | **fridge** BLE protocol (verified against a real capture) |
| `POWER_PROTOCOL.md` | power-station (Fossibot) Modbus protocol |
| `manifest.json` · `service-worker.js` · `icon.svg` | PWA install + offline |
| `task.md` | roadmap |

## Fridge protocol — the things that bite (read before touching `Fridge`)

The fridge uses `FE FE | len | payload | sum16` frames on service `1234` (write `1235`,
notify `1236`). **Full, verified details are in [`PROTOCOL.md`](PROTOCOL.md).** The traps:

1. **Status payload is 30 bytes after the cmd byte, not 31.** `parseFrame` strips the cmd, so
   the dual-zone parse / settings reconstruction must gate on `>= 30`. A `>= 31` check (the
   original bug) silently drops every status into a fallback path and the app looks **stuck
   in pairing**.
2. **Do not auto-send `BIND` (0x00) on connect.** The official app sends `QUERY` (0x01)
   immediately and never binds. Auto-binding is what hung the dual-zone unit. Keep Bind as a
   manual button only.
3. **Chunk every write into ≤20-byte ATT packets.** The official app always does, regardless
   of negotiated MTU; the fridge reassembles by the `FE FE`/length header. A single oversized
   write (the 31-byte settings frame) is silently dropped on a 20-byte link, so settings
   changes appear to do nothing. `Fridge.send()` handles this — keep it.
4. **Temperature commands take ONE byte, not the settings block.** Fridge target =
   `05 <temp>`, freezer target = `06 <temp>`. Both signed int8 in the *displayed* unit.
5. **Settings block (`02`) is the status with the 5 dynamic bytes removed** — see the exact
   index mapping in `PROTOCOL.md` §5. Rebuild it from the latest status, flip one byte, resend.
6. **Units:** setpoints (idx 4 & 18) are stored in the displayed unit; current temps are
   reported in °C and converted for display when °F is active.

## Working agreements

- **Verify against the capture, don't guess.** When changing frame/parse logic, replay it
  against the known-good bytes in `PROTOCOL.md` (a small Node replay of the build/parse/
  reconstruct/chunk path is the fastest check — last run: 13/13).
- **Keep it single-file and dependency-free.** No frameworks, no build tooling.
- **Power-station writes are safety-gated.** Respect the whitelist; `reg 68 = 0` is
  hard-blocked (it can brick the unit). Don't loosen these without a clear reason.
- **It's food/medicine + a battery that can brick.** Be conservative; surface uncertainty in
  the UI/log rather than showing confident-but-wrong values.
- Use **Diagnostics → Snapshot** (raw byte table + change recorder) to confirm any uncertain
  byte offset on real hardware.

## Status

Connection/sync and the dual-zone byte map are verified from a first-party `btsnoop_hci.log`.
The current-temperature offsets in the dynamic tail (status idx 16 & 26) are best-effort —
confirm on hardware if they look off.
