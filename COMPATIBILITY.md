# Compatible fridges — Alpicool & its rebrands

This app speaks the **Alpicool "CAR FRIDGE FREEZER" BLE protocol** (GATT service `1234`,
`FE FE … sum16` frames — see [`PROTOCOL.md`](PROTOCOL.md)). The same fridge hardware is sold
under *many* brand names, so this app should work with a lot more than just Alpicool.

> **The one reliable test.** Brand names don't guarantee anything — a company can rebrand
> Alpicool *hardware* but fit a different Bluetooth module/app. What matters is the **protocol**:
>
> 1. Does the fridge pair with the official **"CAR FRIDGE FREEZER"** app
>    (`com.alpicoolneutral.fridge.controller`, by Alpicool)? **and**
> 2. In this app's **Diagnostics → GATT discovery**, does it list service **`1234`** with
>    write `1235` / notify `1236`?
>
> If both are yes, it's this protocol and this app should drive it. If the app is a *different*
> one, or GATT shows the generic `ffe0/ffe1` (HM‑10) or some other service, it's a different
> lineage and won't work as-is.

Advertised BLE name usually starts with one of: **`A1-`, `AK1-`, `AK2-`, `AK3-`, `WT-`, `K25`, `BC…`**.
(The unit this repo was built against advertises as `A1-FFFF11A96263` — an **Alpicool 50 L
12 V dual-zone, LG compressor**.)

---

## ✅ Confirmed (same app + protocol)

| Brand | Notes |
|:------|:------|
| **Alpicool** | The OEM/original. The 50 L dual-zone here is an Alpicool. |
| **Brass Monkey** | AU/NZ rebrand (Jaycar, Bunnings, Road Tech Marine). Same hardware + same app — confirmed by the `klightspeed/BrassMonkeyFridgeMonitor` and `johnelliott/alpicoold` projects. |
| **BougeRV** (CR-series) | The official app store listing explicitly names BougeRV. ⚠️ *Some* BougeRV models use a different (Wancool/SECOP) compressor + app — check via the test above. |

---

## 🟡 Reported Alpicool-OEM / same-platform rebrands (verify with the test above)

These are widely described as Alpicool-made or sharing the same portable-fridge platform and
are *likely* to use the `1234`/`CAR FRIDGE FREEZER` stack — but confirm per model, because
specs (and the BLE module) vary by production run:

**Setpower · JoyTutus · Bodega · Euhomy · Vevor · Costway · Aspenora · Ausranvik · AstroAI ·
Kalamera · Domende** (and other budget Amazon car-fridge brands).

> These brands are commonly cross-listed as accessory-compatible (cords, seals) with Alpicool,
> which is a strong hint of shared hardware — but accessory compatibility ≠ BLE compatibility.
> The GATT `1234` check is the only proof.

---

## ❌ Different ecosystem (won't work with this app)

Different controllers/apps/protocols — not this `1234` stack:

- **ICECO** (and ICECO-built **Setpower** models) — own app.
- **ARB**, **Dometic**, **National Luna**, **Engel** — entirely different electronics.
- **EcoFlow Glacier**, **Anker** etc. — their own ecosystems.
- Any unit that pairs with a *different* branded app, or exposes generic **`ffe0/ffe1`** (HM‑10)
  rather than service `1234`.

---

## Found another that works (or doesn't)?

Open an issue with: the **brand/model**, the **advertised BLE name prefix**, and a copy of the
**Diagnostics → GATT discovery** output. That's enough to confirm the lineage and grow this list.

## Sources

- Official app — *CAR FRIDGE FREEZER* (`com.alpicoolneutral.fridge.controller`),
  [Google Play](https://play.google.com/store/apps/details?id=com.alpicoolneutral.fridge.controller)
- [klightspeed/BrassMonkeyFridgeMonitor](https://github.com/klightspeed/BrassMonkeyFridgeMonitor) — Alpicool/Brass Monkey, same protocol
- [johnelliott.org — Reverse Engineering a Bluetooth Fridge](https://johnelliott.org/blog/reverse-engineering-a-bluetooth-fridge/) / [johnelliott/alpicoold](https://github.com/johnelliott/alpicoold)
- Brass Monkey Bluetooth pairing help: [Jaycar](https://help.jaycar.com.au/hc/en-us/articles/34829598596889-How-to-pair-Bluetooth-with-a-Brass-Monkey-fridge), [Road Tech Marine](https://help.roadtechmarine.com.au/hc/en-us/articles/35117738038681-How-to-pair-Bluetooth-with-a-Brass-Monkey-fridge)
- [esphome/feature-requests #1375 — Add support for Alpicool Fridge](https://github.com/esphome/feature-requests/issues/1375)

> Compatibility notes are best-effort and community-sourced; the GATT `1234` check is the only
> definitive test. Brand/manufacturer relationships change over time and by region.
