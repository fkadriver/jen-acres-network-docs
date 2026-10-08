# UPS / Power Protection

**Status: planning — not yet purchased.** Power budget below is derived from actual
hardware specs in `~/git/nixos` and the switch config doc, not guesses. Pairs with
[11_CLOSET_COOLING.md](11_CLOSET_COOLING.md) — same closet, same rack.

## Power Budget

| Device | Hardware | Est. load |
|---|---|---|
| nas01 | Dell PowerEdge T330, Xeon E3-1270 v5 (80W), 5 spinning drives (1×500GB + 3×4TB + 1×18TB) | **70W measured** via iDRAC `racadm getsensorinfo` (2026-10-07) — System Board Pwr Consumption 70W, PS1 0.6A/120V≈72W, PS2 idle/standby (confirms redundant PSUs, only one active). Original 120–180W estimate was too high; this reading is at low/idle-ish load (CPU 37°C, fan at 20% PWM, only 1 of 4 DIMM slots populated) — expect it to climb somewhat under a ZFS scrub or simultaneous drive spin-up, but 70W is a solid real-world anchor rather than a guess |
| vm01 | Dell Latitude E7270, headless | ~15–20W |
| OPNsense router | Protectli FW41 | ~6W idle (documented spec, see [01_OPNSENSE_INSTALLATION.md](01_OPNSENSE_INSTALLATION.md)) |
| log01 | Shuttle Zingbox, Celeron (Jasper Lake-class) | ~10–15W |
| Aruba 2530-24G switch + 2× UniFi U6-Pro APs | Switch baseline + 42W of 195W PoE budget actually used (see [04_SWITCH_CONFIG.md](04_SWITCH_CONFIG.md)) | ~70–85W |
| sands-bak01 | HP ProDesk 600 G4 Desktop Mini | ~15–25W |
| 15" Compaq monitor | LCD, confirmed | ~20–30W |
| pihole01 + pihole02 | 2× Raspberry Pi 3B | ~5W each, ~10W total |

**Typical running total: ~205–255W** (nas01 now measured rather than estimated).
**Realistic peak (nas01 under scrub/spin-up + everything else at the high end):
~350–450W.**

## Sizing

Rule of thumb: size for ~1.5–2× real running load, not peak — headroom for inrush
current (HDD spin-up draws a brief surge above steady-state), keeps the UPS from
running near its ceiling continuously, and buys more runtime at actual load than the
rated-capacity runtime number implies. ~230W typical load → **800–1200VA / 500–700W**
would size it tightly; the recommendation below sits a notch above that for extra
runtime margin and future load growth.

## Two things that matter more than capacity

1. **Pure sine wave output, not simulated/stepped.** nas01's Dell PSU (and most
   enterprise server PSUs) use active PFC, which can misbehave, buzz, or refuse to run
   on a simulated-sine-wave UPS once it switches to battery. Non-negotiable here.
2. **NUT (Network UPS Tools) compatibility** — needed for the Home Assistant
   integration (battery %, load wattage, runtime, clean "on battery" / shutdown
   signal). APC's `usbhid-ups` driver support is the most battle-tested; CyberPower's
   PFC-series also works well via the same driver.

Do **not** put the UPS or anything behind it on a switchable smart plug — see the power
safety note in [11_CLOSET_COOLING.md](11_CLOSET_COOLING.md).

## Recommendation

**CyberPower CP1500PFCLCD** (1500VA/900W, pure sine wave, LCD status panel, 10
outlets — 6 battery+surge, 4 surge-only, ~$200) — now comfortably oversized relative to
the measured ~230W typical load (good — more runtime margin and room for future
growth), well-supported under NUT, and the 10 outlets cover all 8 devices without a
power strip in the mix. Tower form factor, fits the wire shelving better than a
rack-mount unit.

Alternative with a more proven NUT track record, at a price step-up: **APC Smart-UPS
SMC1500** or **SMT1500** (1500VA/900W, pure sine wave, ~$400–500) — the
homelab-community gold standard for USB/NUT monitoring.

## Outstanding / TODO

- [x] Pull nas01's live power draw from iDRAC — 70W measured, 2026-10-07
- [x] Confirm whether the 15" Compaq monitor is LCD or CRT — confirmed LCD
- [ ] Purchase UPS (CyberPower CP1500PFCLCD or APC Smart-UPS SMC1500/SMT1500)
- [ ] Wire into the new 20A circuit (see [11_CLOSET_COOLING.md](11_CLOSET_COOLING.md))
- [ ] Set up NUT (HA add-on or small host) talking to the UPS over USB
- [ ] Confirm "on battery" / low-battery signal reaches Home Assistant and triggers a
      graceful nas01 shutdown on extended outage
