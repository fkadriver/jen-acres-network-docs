# UPS / Power Protection

**Status: ordered — arriving Friday 2026-11-27.** Power budget below is derived from actual
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
| Starlink Gen 5 dish + Gen 3 router | Dish is outdoors, powered over its cable from the router's PSU in the closet — the whole kit draws through one closet outlet | Dish **21W measured** (Starlink app, 15-min average, 2026-11-15). The app reports dish power only, so add an estimated ~10–15W for the Gen 3 router: **~30–40W typical**. Snow-melt heating will push it much higher; budgeted at ~100W peak until there's a winter reading |
| T-Mobile 5G gateway | Tier2 cellular backup WAN ([10_STARLINK_MULTIWAN.md](10_STARLINK_MULTIWAN.md)) | **~10–15W typical** (T-Mobile publishes 10–14W ready-state across its gateway models), 45W theoretical max |

**Typical running total: ~245–310W** (nas01 and the Starlink dish measured). **Realistic peak (nas01 under scrub/spin-up, Starlink heating,
everything else at the high end): ~500–600W.**

The existing 450VA UPS (~250–280W) can't carry this. Typical load alone is at or over
its rating, and it's almost certainly simulated sine wave (see below).

## Sizing

Rule of thumb: size for ~1.5–2× real running load, not peak — headroom for inrush
current (HDD spin-up draws a brief surge above steady-state), keeps the UPS from
running near its ceiling continuously, and buys more runtime at actual load than the
rated-capacity runtime number implies. ~280W typical load → **~1000VA / 600W**
would size it tightly. 1500VA / 900W keeps peak load near 65% and gives roughly
20 minutes of runtime at typical load.

## Two things that matter more than capacity

1. **Pure sine wave output, not simulated/stepped.** nas01's Dell PSU (and most
   enterprise server PSUs) use active PFC, which can misbehave, buzz, or refuse to run
   on a simulated-sine-wave UPS once it switches to battery. Non-negotiable here.
2. **NUT (Network UPS Tools) compatibility** — needed for the Home Assistant
   integration (battery %, load wattage, runtime, clean "on battery" / shutdown
   signal). APC's `usbhid-ups` driver support is the most battle-tested; CyberPower's
   PFC-series also works well via the same driver.

## Outlet / power strip layout

Keep the existing **network power strip**: one button power-cycles the whole
network stack and leaves the servers alone. That strip plugs into a single UPS battery
outlet, which is also what makes the outlet count work.

| UPS outlet | What's on it |
|---|---|
| Battery 1 | **nas01 PSU1**, plugged in directly (no strip) |
| Battery 2 | **nas01 PSU2**, plugged in directly (optional: same UPS, so it only protects against a PSU failure, not a power-feed failure) |
| Battery 3 | **Network strip** (the reset button): Aruba switch (+ AP PoE), OPNsense, Starlink router/PSU, T-Mobile gateway, Pi-holes. ~125–155W typical, up to ~250W at peak |
| Battery 4 | **Server strip**: vm01, log01, sands-bak01. ~40–65W |
| Battery 5–6 | Spare |
| Surge-only | 15" Compaq monitor (doesn't need battery) |

Notes:
- Strips plugged into the UPS should be **plain power strips, not surge protectors**.
  UPS makers (APC and CyberPower) say not to chain a surge suppressor behind a UPS:
  the suppressor's MOVs can fight the UPS's output on battery, and doing it can void
  the UPS's equipment-protection warranty. The UPS already provides surge protection.
- Power both Pi-holes from one dual-port USB charger on the network strip, so they
  take one strip outlet instead of two.
- A network-strip reset hard-cuts power to the Pis. The SD-card corruption risk is
  the same as today, so it's no reason to change the setup.
- nas01 and its NUT shutdown signal aren't on any reset strip, so a network reset
  never hard-cuts the ZFS box.

### Option B: two UPSes (keep the existing 450VA for WAN gear)

The layout above is **Option A**: everything on the one 1500VA UPS. That's the simplest
setup, and the network outlasts or matches the servers.

If the existing 450VA (~250–280W) stays in service, split by what the graceful
shutdown depends on, not by "network vs servers". The switch and OPNsense carry the
NUT shutdown signal and the LAN, so they stay on the big UPS with the servers. Only
the WAN gear goes on the 450:

| UPS | What's on it | Est. load |
|---|---|---|
| 1500VA | nas01 (direct), server strip (vm01, log01, sands-bak01), **core network strip** (Aruba switch + AP PoE, OPNsense, Pi-holes) | ~205–255W typical → ~20–25 min |
| 450VA | **WAN strip**: Starlink router/dish, T-Mobile gateway | ~40–55W typical, ~145W snow-melt peak → roughly 25–40 min on a healthy battery |

Trade-offs:
- The LAN stays up as long as the servers do, so NUT can always deliver the
  shutdown. Internet may drop first if the 450 runs out, but shutdown doesn't need it.
- The 450 ends up lightly loaded and gets decent runtime, unlike putting the whole
  network stack on it (~125–155W typical, ~250W peak, near its limit).
- **The one-button reset splits in two.** Cycling the core strip resets the switch,
  OPNsense and the Pi-holes. Starlink and T-Mobile need their own strip cycled. That's
  usually fine, because modem/WAN resets and LAN resets are different problems.
- A 450VA is almost certainly simulated sine wave. That's fine for the Starlink and
  T-Mobile power adapters; only nas01's active-PFC PSU needs pure sine.
- Replace the 450's battery if it's more than ~3 years old.

**For either option:** run the NUT server on **nas01 itself**, with the UPS USB cable
plugged into it, so its own shutdown never depends on the network. Other hosts (vm01,
log01, sands-bak01, Home Assistant) connect to it as NUT clients over the core
network, which stays powered as long as they are.

Do **not** put the UPS or anything behind it on a switchable smart plug — see the power
safety note in [11_CLOSET_COOLING.md](11_CLOSET_COOLING.md).

## Recommendation

**CyberPower CP1500PFCLCD** (1500VA/900W, pure sine wave, LCD status panel, 10
outlets — 6 battery+surge, 4 surge-only, ~$200) — about 30% loaded at the ~280W typical
load, well supported under NUT, and its 6 battery outlets fit the layout above with
2 to spare. Tower form factor, fits the wire shelving better than a
rack-mount unit.

Alternative with a more proven NUT track record, at a price step-up: **APC Smart-UPS
SMC1500** or **SMT1500** (1500VA/900W, pure sine wave, ~$400–500) — the
homelab-community gold standard for USB/NUT monitoring.

## Outstanding / TODO

- [x] Pull nas01's live power draw from iDRAC — 70W measured, 2026-10-07
- [x] Confirm whether the 15" Compaq monitor is LCD or CRT — confirmed LCD
- [x] Read Starlink dish draw from the app: 21W average, 2026-11-15
- [ ] Grab a winter Starlink reading while snow-melt is running to replace the ~100W peak estimate
- [ ] Swap any surge-protector strips that will sit behind the UPS for plain power strips
- [ ] Decide Option A (single 1500VA) vs Option B (1500VA + existing 450VA for WAN)
- [x] Purchase UPS (CyberPower CP1500PFCLCD or APC Smart-UPS SMC1500/SMT1500) — ordered 2026-11-24, arriving Friday 2026-11-27
- [ ] <!-- todo: Install closet UPS --> Wire UPS and rack into power (UPS arrives 2026-11-27)
- [ ] Set up NUT (HA add-on or small host) talking to the UPS over USB
- [ ] Confirm "on battery" / low-battery signal reaches Home Assistant and triggers a
      graceful nas01 shutdown on extended outage
