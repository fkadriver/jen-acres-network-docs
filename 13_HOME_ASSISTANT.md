# Home Assistant / IoT Network

**Status: radios in progress.** Zigbee (ZBT-2, channel 25) and two ESPHome Bluetooth
proxies (office, garage) are live; Z-Wave (ZWA-2) not yet purchased. This doc covers
the HA platform itself and device/protocol decisions across the whole house. Device-specific implementation for the server closet (temp sensor, fan
plug) lives in [11_CLOSET_COOLING.md](11_CLOSET_COOLING.md); it just consumes the
Zigbee mesh set up here.

## Hardware: HA Green

HA Green has **no built-in radios** — Zigbee, Thread, Z-Wave, and Bluetooth all require
external USB adapters. Currently sits on top of the metal cabinet on the left side of
the server closet — not enclosed in metal, which helps, but still a 2.4GHz-noisy
environment (WiFi APs, multiple WiFi-connected devices, metal shelving nearby). Radio
adapters should be on USB extension cables routed out of the closet where practical
(see the closet doc's Placement section for the physical routing plan).

## Zigbee vs Z-Wave

| | Zigbee | Z-Wave |
|---|---|---|
| Band | 2.4GHz — shares spectrum with WiFi/Bluetooth | Sub-GHz (908.42MHz US) — no WiFi contention |
| Interop | Vendor extensions common (e.g. Aqara quirks on ZHA with few routers) | Z-Wave Alliance certifies all devices — stricter guaranteed interop |
| Device cost/selection | Cheap, huge ecosystem (sensors, plugs, bulbs) | Pricier, smaller ecosystem, but mature in security/mains-switching categories |
| Range per hop | Shorter | Longer; Z-Wave Long Range claims near-mile in open conditions |
| HA official adapter | Connect ZBT-2 (~$49) | Connect ZWA-2 (~$69) |
| Best fit here | High-volume cheap sensors/plugs (temp/humidity, smart plugs) | Door locks, in-wall light switches — categories where certified reliability matters more than price |

**Zigbee (ZBT-2): confirmed need now**, for the high-volume sensor/plug fleet — same
pattern as the closet's SNZB-02D + smart plug. **Z-Wave (ZWA-2): justified eventually,
via Z-Wave Long Range, but not an immediate purchase** — the barn (see Barn Lighting
below) is a 2–3 year want, not a current requirement, so there's no rush on this radio.
Not because of the door lock either (see Door Locks below, that's no longer a forcing
function). Worth knowing the answer in advance, though: Zigbee has no realistic path to
100 yards of open acreage short of a chain of mains-powered repeaters with nothing to
plug them into along the way; Z-Wave LR's star topology (direct hub↔device, no mesh
hops, rated to 1300 ft) is purpose-built for exactly this
"detached building on a large property" case.

## Radio Adapters

### Zigbee: Home Assistant Connect ZBT-2

Official successor to the ZBT-1 (now discontinued). Zigbee/Thread/Matter USB adapter,
~$49.

- [Official product page](https://www.home-assistant.io/connect/zbt-2/)
- [Ameridroid](https://ameridroid.com/products/home-assistant-connect-zbt-2)
- [Everything Smart](https://shop.everythingsmart.io/products/home-assistant-connect-zbt-2)

Cheaper alternative: **Sonoff ZBDongle-E** (~$18–30, same Silicon Labs EFR32MG21 chip)
— [Amazon](https://www.amazon.com/sonoff-zigbee-dongle/s?k=sonoff+zigbee+dongle),
[AliExpress](https://www.aliexpress.us/item/3256803932977988.html).

Favor Sonoff/Third Reality Zigbee devices over Aqara initially — Aqara has had known
pairing quirks on a ZHA mesh with few routers present.

### Z-Wave: Home Assistant Connect ZWA-2

Silicon Labs ZG23 (Z-Wave 800 series), supports standard Z-Wave and Z-Wave Long Range
simultaneously, ~$69.

- [Official listing — Ameridroid](https://ameridroid.com/products/home-assistant-connect-zwa-2)
- [Pimoroni](https://shop.pimoroni.com/en-us/products/home-assistant-connect-zwa-2)
- [The Pi Hut](https://thepihut.com/products/home-assistant-connect-zwa-2)

### Bluetooth: no dedicated radio — ESPHome proxies instead

No single BT adapter on the HA Green host would cover the house — range is short and
the closet's RF environment is poor anyway. Use **ESPHome Bluetooth proxies** (cheap
ESP32 boards flashed with proxy firmware) placed per room/area as BLE device coverage
expands.

**Live now (2026-11-09): two proxies placed.**

- **Office** — **M5Stack Atom Lite** (~$7.50–10; ESP32 + FTDI FT232 USB-serial
  chip). [M5Stack store](https://shop.m5stack.com/products/atom-lite-esp32-development-kit) /
  [Amazon](https://us.amazon.com/M5Stack-Official-Atom-ESP32-Development/dp/B0FB94ZFXY) /
  [Pimoroni](https://shop.pimoroni.com/products/atom-lite-esp32-development-kit) /
  [The Pi Hut](https://thepihut.com/products/atom-lite-esp32-development-kit).
  Registered in HA as both an `esphome` config entry and a `bluetooth` scanner
  entry (`office-ble-proxy`, B8:7B:4D:18:EB:EA).
- **Garage** — a cheap AliExpress-style board sold as "ESP32-C3 SuperMini," but
  **esptool confirmed it's actually an ESP32-C6** at flash time — a known
  mislabeling pattern on these clone boards, so trust what esptool reports over
  the listing title. Uses the chip's native USB-JTAG/serial controller
  (`idVendor:idProduct` = `303a:1001`), a different USB identity from the
  office board's FTDI chip. Registered in HA as `esphome` + `bluetooth`
  scanner entries (`garage-ble-proxy`, 90:64:9B:AA:5A:24/26).

Neither board flashes via ESPHome's browser-based web installer here — the
daily-driver browser (Firefox) doesn't support the WebSerial API it needs.
Instead: `~/git/nixos/modules/esphome-flash.nix` (NixOS module, enabled on
`latitude`) installs the `esphome` CLI plus udev rules mapping either board's
USB-serial chip to a stable `/dev/esp32-flash` symlink (one rule per chip
type, both boards land on the same symlink name since only one is flashed at
a time), and a wrapper script (`esphome-flash <config.yaml>`) that points
`esphome run` at it. ESPHome configs and WiFi/API secrets live in `~/esphome/`
on `latitude`, outside any repo — not git-tracked, since it holds the WiFi
password and per-device API encryption keys.

HA Green's own host also picked up a plugged-in USB Bluetooth dongle
(Realtek), registered as a third `bluetooth` scanner — not an ESPHome proxy,
just the host's local radio adding its own coverage near wherever HA Green
physically sits.

Budget one proxy per room/area with actual BLE devices, not one trying to cover the
whole house.

## 2.4GHz Coexistence (WiFi / Zigbee / Thread)

Worth planning for, not worth losing sleep over.

**The overlap:** WiFi 2.4GHz uses channels 1/6/11 (the only non-overlapping set,
~2412–2472MHz). Zigbee and Thread both run on 802.15.4, channels 11–26
(2405–2480MHz) — channels 25–26 sit above WiFi channel 11's upper edge, mostly clear
of a standard 1/6/11 WiFi deployment. Solvable with channel planning, not an inherent
conflict.

**Why it's a smaller deal here specifically:**

- The UniFi APs are wired via PoE (ports 13/14, see
  [04_SWITCH_CONFIG.md](04_SWITCH_CONFIG.md)), not wireless-backhauled — no mesh
  backhaul traffic competing for 2.4GHz airtime the way a true wireless mesh adds.
- Zigbee/Thread's actual bandwidth need is tiny — short periodic sensor reports and
  on/off commands, not streaming — tolerates shared-band noise far better than
  continuous traffic would.

**Actions:**

1. Lock the UniFi 2.4GHz radios to fixed channels 1, 6, or 11 (not Auto), so the plan
   below has a known target to avoid.
2. Set the Zigbee coordinator's PAN to **channel 25** explicitly at setup (ZHA prompts
   for this). Pick it once — changing it later means re-pairing every device, since
   Zigbee devices don't roam channels on their own.
3. Physical separation (the USB-extension-out-of-the-closet plan above) matters more
   than channel choice alone — distance from the APs and the metal shelving cuts
   interference more than spectrum placement does.

### Thread — findings/thoughts

Not in active use yet, and nothing currently on the device list needs it — this is
groundwork for later, not a near-term purchase.

- **What it actually is**: an IP-based (6LoWPAN) mesh protocol, unlike Zigbee's
  non-IP mesh — Thread devices get routable IPv6 addresses. It's the mesh transport
  Matter rides on, not a competitor to Zigbee so much as a different plumbing layer
  for newer devices.
- **Needs a Border Router**: something has to bridge the Thread mesh to the rest of
  the IP network. Common sources: Apple HomePod mini/TV 4K, Google Nest Hub (2nd
  gen)/Nest WiFi Pro, Amazon Eero (4th gen+)/Echo (4th gen+) — or Home Assistant's own
  **OpenThread Border Router (OTBR)** add-on, which needs its own dedicated Thread
  radio.
- **The ZBT-2 can't do both at once.** It runs Zigbee *or* Thread, not
  simultaneously (no MultiPAN support in the current firmware) — since it's already
  committed to Zigbee here, a Thread border router later means either a second
  dedicated radio (another ZBT-2 or similar, ~$49 more) or relying on a border router
  from an existing Apple/Google/Amazon device if one's already in the house.
- **Tension with the deconsolidation goal**: leaning on an Apple/Google/Amazon device
  as the Thread border router works, but cuts against the broader push to reduce
  big-tech dependency in the home stack. A second dedicated radio running HA's own
  OTBR is the more local-first option if/when Thread actually becomes necessary.
- **If it does get deployed**: give it its own channel, separate from the Zigbee
  network's channel 25 — two 802.15.4 networks on the same or adjacent channel
  interfere with each other the same way either one does with WiFi. Channel 15 is a
  reasonable pick (clear of WiFi, separated from Zigbee's 25).
- **Trigger to revisit**: the first time a Matter-over-Thread device actually shows
  up on the shopping list (several newer BT-light and sensor lines are shipping
  Thread variants) — no reason to provision for it before then.

## Device Categories

### Weather Station — needs a pick

Outdoor sensor→gateway is always proprietary sub-GHz regardless of brand (not a
Zigbee/Z-Wave choice); the gateway→HA hop is local LAN either way, not cloud, if
configured right. Two solid options:

- **Ecowitt** — cheaper, modular (add rain/wind/soil/UV sensors independently), fully
  local via the **Ecowitt Local** integration (HACS, no custom repo needed) — polls the
  gateway's local API directly, no cloud, no webhook exposure. Good fit for the
  no-cloud preference.
- **WeatherFlow Tempest** — pricier (~$329), no moving parts (sonic anemometer, no
  spinning cups/vanes to fail), near-zero maintenance. Core HA integration is 100%
  local (reads the hub's UDP broadcast on the LAN, no account needed) but only exposes
  the raw sensor set; the optional WeatherFlow Cloud integration adds forecast data if
  wanted later.

Lean Ecowitt if cost/modularity matter more; Tempest if "install once, never touch
again" is worth the premium.

### Door Locks

**Confirmed (2026-10-xx):** the installed lock is the original **ULTRALOQ U-Bolt**,
purchased 2021 — Amazon listing confirms "5 in 1 Keyless Entry... with Bluetooth, App
Control" with no mention of WiFi or Z-Wave. This is the **Bluetooth-only** base model,
not the newer Z-Wave-certified U-Bolt Pro/Bolt lines. Confirmed via research: **there
is no Z-Wave module or upgrade path for this generation of lock** — Z-Wave Ultraloqs
are a separate product line, not a retrofit. Getting Z-Wave here means replacing the
lock outright, not adding a module.

Two real paths, not one:

1. **Keep the lock, go local-BLE.** Community integrations
   ([ultraloq-ble-ha, chaseflick4 fork](https://github.com/chaseflick4/ultraloq-ble-ha) —
   explicitly supports the original U-Bolt;
   [inventor7777 fork](https://github.com/inventor7777/ultraloq-ble-ha) is the broader
   alternative) give local Bluetooth control in HA — no cloud round-trip for day-to-day
   lock/unlock, internet only needed once during initial setup (pulls lock info from
   U-tec's Xthings API). Needs active GATT Bluetooth reachable from HA — the host's own
   radio, a USB adapter, or (the natural fit here) **an ESPHome Bluetooth proxy near the
   front door** — same infrastructure already planned for the BT lights, just another
   proxy placement. **Zero hardware replacement cost.** Caveat: this is a
   community-maintained integration (two competing forks exist, and the Ultraloq
   community forum has an open thread asking for better official support) — worth
   weighing that maturity level against the fact that it's controlling the front door,
   not a light bulb.
2. **Replace with an official Z-Wave Ultraloq** (U-Bolt Pro Z-Wave or the newer Bolt
   Z-Wave) — officially "Works with Home Assistant" certified, fully local, no
   community-integration dependency. Costs a new lock (~$180–220+) to get there.

**This removes the lock as automatic justification for buying the ZWA-2** — path 1
keeps everything on Zigbee + Bluetooth infrastructure already planned, no Z-Wave radio
required for the door at all. Doesn't matter either way now that the barn (below)
justifies the ZWA-2 on its own, but still the right call for the lock specifically on
its own merits.

### Barn Lighting (100 yds, steel construction) — want, 2–3 years out

20+ acre property, steel barn ~100 yards from the house. Only ask here is **light
control** — not sensors, not security, not a full automation buildout. **Not a current
requirement** — captured now because it settles the Zigbee-vs-Z-Wave question in
advance, not because anything needs buying yet.

- **Z-Wave Long Range, not Zigbee.** This is the concrete case that justifies the
  ZWA-2 (see Zigbee vs Z-Wave above) — Z-Wave LR's star topology reaches a detached
  building at this distance directly, no mesh chain required.
- **Steel is the harder problem, not distance.** A steel building attenuates RF
  severely regardless of protocol — getting a signal to the barn's exterior wall and
  getting it *into* the structure to a switch mounted inside are two different
  problems. Mount the switch as close to a door/window/eave opening as practical
  rather than deep inside the structure, and treat the published 1300 ft Z-Wave LR
  range figure as a best case, not a guarantee, given the steel.
- **Hardware**: a Z-Wave LR in-wall switch at the barn's existing light switch
  location — **Zooz ZEN76 800LR** (on/off, ~$35–40) for simple control, or **ZEN77
  800LR** (dimmer, ~$45–50) if dimming is wanted. Both need the ZWA-2 (or a Zooz
  ZST39 LR stick) as the controller, with Long Range mode enabled in Z-Wave JS.
- **Test before committing.** Bring a single LR device out to the barn's actual switch
  location and verify a usable signal before buying more than one — 100 yards through
  open air is well within spec, but the steel construction is the real unknown here,
  not the distance.
- **Fallback if Z-Wave LR doesn't reach reliably**: a dedicated point-to-point outdoor
  WiFi bridge (e.g. Ubiquiti airMAX/NanoStation pair, one at the house, one at the
  barn) extends the actual LAN out there, after which any ordinary WiFi smart switch
  works locally at the barn end — a bigger project, but fully reliable once in place,
  and opens the door to anything else out there later (cameras, sensors) if that ever
  comes up.

### Temp/Humidity Sensors (whole-house + future per-room fan units)

Not centrally controlling house temp — this is monitoring, plus possibly repeating the
closet's pattern (sensor + Generic Thermostat helper + fan plug) for other rooms later.
**Zigbee** — same SONOFF SNZB-02D pattern as the closet, cheap and battery-friendly,
reuses the ZBT-2 mesh already being built out.

### BT Lights (office now, expanding)

Bluetooth, not a dedicated radio — covered by ESPHome proxies as described above. As
this expands beyond the office, add a proxy per new room rather than relying on range
from the existing one. Worth considering **Zigbee bulbs for new purchases** going
forward (IKEA Trådfri, Sengled, etc.) — they integrate more natively/reliably with HA
than BLE and don't depend on proxy coverage, while the existing office BT lights stay
as-is.

### Remote Light Switches

**Z-Wave in-wall switches** (Zooz, Inovelli, GE/Jasco) are the reliability standard for
mains switching — same certified-interop argument made for locks above. The ZWA-2 is
eventually coming regardless (see Barn Lighting above), but that's 2–3 years out, so
don't let it drive an in-house switch decision that's needed sooner — Zigbee alternatives exist
(Third Reality, Sonoff ZBMINI, IKEA) if keeping in-house switches on the cheaper/larger
ecosystem is preferred anyway — no longer a decision that determines whether a Z-Wave
radio gets bought at all, just which protocol these specific switches land on.

### Baseboard Heater Thermostats

Replacing old **Cadet T410B111** mechanical double-pole line-voltage thermostats
(22A @ 240V rating — just the switching ceiling, not the actual heater load) with
Zigbee control, same Zigbee-first reasoning as everywhere else in this doc.

**Primary solution: Zigbee relay + spare SNZB-02D + Generic Thermostat helper** —
reuses sensors already purchased (two SNZB-02D bought for the closet, one spare)
and the exact same pattern already built for the closet fan (see
[11_CLOSET_COOLING.md](11_CLOSET_COOLING.md)), instead of paying for a thermostat
with built-in sensing that goes unused.

- **SONOFF MINI-ZB1GS ("MINI ONE")** — Zigbee 3.0, rated 100–240V / 16A max,
  ~$18–20. Sonoff's Zigbee lineup has paired cleanly with ZHA every other time in
  this project (S40 Lite, ZBDongle-E) — no quirk-compatibility question mark the
  way Stelpro had, though no dedicated ZHA quirk was found for this specific model
  either (not unusual for Sonoff switches — most just work via the standard Zigbee
  On/Off cluster without needing one).
- **Correction: this is a "neutral-required" device, not a no-neutral one** — SONOFF's
  "ZBMINI Extreme" (ZBMINIR2) naming initially looked like the no-neutral option,
  but checking SONOFF's own spec pages directly: **ZBMINIR2 requires a neutral**
  (10A/2400W @ 240V); the actual no-neutral model is the different, lower-rated
  **ZBMINIL2** (6A). The MINI-ZB1GS is also a neutral-required design — but that's
  fine here, explained below.
- **Why a "neutral-required" relay still works on a true no-neutral 240V
  split-phase circuit**: the device doesn't actually need a *true* neutral, just a
  voltage differential within its rated range across its L and N terminals to power
  itself — and unlike a lighting retrofit where the switch box is genuinely missing
  a second conductor, this box already has **two hot legs physically present**
  (L1, L2). Wire **L1 → its "L" terminal** (powers the device and feeds the relay),
  **L2 → its "N" terminal** (self-power reference only, not part of the switched
  path) — same 240V differential a true hot+neutral circuit would give it. The
  relay's switched output goes to one heater leg; L2 also gets pigtailed straight
  to the heater's other leg (unswitched), matching the single-pole-is-fine
  conclusion below.
- **Amperage check**: Cadet 4F1000 heater draws 4.2A continuous; relay's rated
  16A — only **26% of capacity**, well under the usual 80% continuous-load
  guideline, and it's a purely resistive load (no inrush/arcing concern the way a
  motor would add). Comfortable margin.
- **Dry-contact caveat doesn't apply here**: some baseboard heaters have their own
  electronic control board needing a dry-contact signal rather than direct
  line-voltage switching (frying the board otherwise, needs a contactor in between).
  The Cadet 4F1000 is a plain resistive element, no control board — direct relay
  switching in series is exactly how the original T410B111 worked.
- Fits in the same single-gang box the Cadet occupies now.
- Pair the relay + one spare SNZB-02D to ZHA, then a **Generic Thermostat helper**
  (sensor = SNZB-02D, switch = relay, **AC mode OFF this time** — unlike the closet,
  this is heating logic, not cooling, so AC mode should stay off/default here).

**Why single-pole switching is fine on this double-pole circuit**: confirmed this
circuit has a **double-pole breaker in the panel** (slots 23/25) — that breaker is
the code-recognized disconnecting means, not the thermostat/relay, and there's no
local code requirement forcing double-pole switching downstream either. Wiring: the
Cadet's double-pole box has two hot wires in and two hot wires out — land both
incoming hots on the relay's line-side terminals per above, but only switch **one**
outgoing hot; the other outgoing hot (to the heater) gets spliced straight to its
matching incoming hot (pigtailed, not switched).

**Fallback: Sinopé TH1123ZB — single-pole, extremely expensive for what it is
(~$80–160 depending on retailer) given it's just a relay + sensor + logic, the same
three things the primary solution above assembles for ~$20.** Keeping the research
since it's the better-documented, zero-assembly option if the relay approach turns
out to have issues:

- Rated 3000W/12.5A @ 240V, confirmed in **ZHA's own quirks module**
  (`zhaquirks/sinope/thermostat.py`) — not just Zigbee2MQTT.
- The double-pole sibling, **TH1500ZB**, is the "correct" match for the Cadet's
  original double-pole wiring, but Amazon currently lists it unavailable and its
  manufacturer US product page 404s — not clearly discontinued, just hard to
  actually buy right now. Moot anyway since single-pole is confirmed fine here.
- If a future heater's load exceeds ~3000W/12.5A @ 240V: step up to **TH1124ZB**
  (same single-pole family, 4000W/16.7A @ 240V).
- Stelpro's Zigbee lineup (SMT402/ASMT402 Maestro, ~$90–100) is cheaper than Sinopé
  but has **no confirmed ZHA quirk** (zero hits searching `zigpy/zha-device-handlers`
  for "stelpro") — real uncertainty on whether manufacturer-specific extras work
  under ZHA specifically, vs. Zigbee2MQTT where it's documented. Not worth the
  savings given the relay approach above is both cheaper and better-understood.
- Wiring if ever used: land both incoming hots on the line terminals, only connect
  one outgoing hot to the load terminal, pigtail the other straight through — same
  single-pole wiring approach as the relay solution above.

### Smoke / CO Detectors

Real combustion risk here, not a hypothetical: **gas furnace, gas water heater, and
a wood fireplace** — three separate CO sources, not just one.

**Picks: Heiman HS1SA-E (smoke) and HS1CA-E (CO)** — separate standalone units, not
a combo box (Heiman doesn't bundle smoke+CO into one device the way
Kidde/First Alert combo alarms do). Both confirmed in **ZHA's own quirks module**
(`zhaquirks/heiman/hs1sa_e_lover.py`, `zhaquirks/heiman/hs1ca_e_plus.py`). Heiman
became an official "Works with Home Assistant" partner in 2026.

**Avoid Aqara's smoke detector** — community reports flag it as too trigger-happy
with steam (false alarms near showers/kitchens), consistent with the Aqara
pairing-quirk caution already noted above for other device categories.

**Safety framing — not a formality**: these are the "also notify Home Assistant"
layer on top of actual life-safety detection, not a replacement for it. If
UL-listed, code-compliant standalone smoke/CO alarms aren't already in place at the
required locations, that's the non-negotiable first layer regardless of what gets
added here.

**Smoke side is already covered** — standalone smoke detectors already exist in
all bedrooms. HS1SA-E units are an opportunistic replace-as-needed upgrade path
(e.g. when an existing one hits end-of-life or starts chirping), not an urgent bulk
purchase. **CO side is the actual gap** — no existing CO detection mentioned, so
the HS1CA-E placement below is net-new, not a replacement.

**Placement — confirmed layout**: furnace and water heater share one basement
utility room (also has an office and an unused bedroom down there); all
currently-used bedrooms are on the main level off a single hallway; fireplace is
also on the main level.

**3 HS1CA-E units, not collapsing the fireplace and hallway into one just because
they're on the same floor** — "near sleeping areas" is a per-location code
requirement (in the immediate vicinity of the bedrooms), not a per-floor one, and a
fireplace placed for combustion-source coverage isn't necessarily close enough to
the bedroom hallway to double as that:

- **Basement utility room (furnace + water heater)** — one HS1CA-E, since both
  appliances share the same room. Earliest-warning location too — catches a
  problem before it diffuses up through the house. Incidentally also covers the
  basement office and unused bedroom, though that's not the placement's primary
  justification (nobody's sleeping there currently).
- **Main level, near the fireplace** — one HS1CA-E, for incomplete
  combustion/damper issues specifically, separate from the furnace/water heater's
  own safety shutoffs.
- **Main level, at/near the bedroom hallway** — one HS1CA-E, the actual
  code-driven sleeping-area placement. Keep this distinct from the fireplace
  unit above unless the fireplace happens to sit directly off that same hallway —
  if so, one unit there could reasonably cover both; otherwise, two separate
  main-level units.

Each pairs independently to ZHA.

### Legacy 433MHz Sensors (Sharp SPC1107 outdoor sensor) — speculative, not a plan

Already own a Sharp SPC1107 atomic wall clock; its external temp/humidity sensor
transmits at **433.92 MHz** — not natively supported by HA. Checked rtl_433's
supported-device list: Sharp's only existing decoder is for the **SPC775**, a
different model at 917.2 MHz FSK — different band and modulation, not compatible
with the SPC1107.

Possible path, if ever pursued: an RTL-SDR USB dongle (~$25–35) + the community
**rtl_433** HA add-on. Decent odds an existing generic 433MHz decoder happens to
match it already (cheap sensors in this band often share one of a handful of OEM
chipsets); if not, reverse-engineering the bit layout from `rtl_433 -A` raw captures
is plausible (the SPC775 decoder source is a clean template for this) but not
guaranteed to work out. An afternoon of SDR tinkering, not a real integration plan —
lower priority than just buying another SONOFF SNZB-02D (same pattern as the closet)
for the same outcome. Only worth doing as a radio-tinkering exercise, not because the
sensor itself is needed.

## Outstanding / TODO

- [x] Purchase and set up Zigbee coordinator — SONOFF ZBDongle-E (ezsp radio), channel
      locked to 25; closet fan plug + temp sensor both paired (see
      [11_CLOSET_COOLING.md](11_CLOSET_COOLING.md))
- [x] Confirm which Ultraloq U-Bolt variant is installed — 2021 U-Bolt, Bluetooth-only,
      no Z-Wave module/upgrade path exists for this generation
- [x] Place ESPHome Bluetooth proxy in the office for the existing BT lights —
      M5Stack Atom Lite, live 2026-11-09
- [x] Place ESPHome Bluetooth proxy in the garage — ESP32-C6 SuperMini-style
      clone board, live 2026-11-09
- [ ] Place a third ESPHome Bluetooth proxy near the front door if the local-BLE
      Ultraloq lock path is chosen (see Door Locks above)
- [ ] Decide lock path: local-BLE community integration (free, unofficial) vs
      replacing with an official Z-Wave Ultraloq (~$180–220+, certified)
- [ ] Decide remote light switch protocol (Z-Wave vs Zigbee) for the in-house switches
      — not gated on the barn/ZWA-2 timeline, that's years out
- [ ] Pick weather station (Ecowitt vs Tempest) and set up its local HA integration
- [ ] Purchase SONOFF MINI-ZB1GS, replace Cadet T410B111 on the Cadet 4F1000
      baseboard heater (single-pole switching confirmed OK — double-pole breaker
      already provides the disconnect; relay rated 16A vs. heater's 4.2A draw)
- [ ] Pair relay + one spare SNZB-02D to ZHA, configure Generic Thermostat helper
      (AC mode OFF — this is heating logic, not the closet's cooling logic)
- [ ] Purchase 3 Heiman HS1CA-E (CO) units — the actual gap, no existing CO
      detection — placement: basement utility room (furnace+water heater), main
      level near fireplace, main level at/near the bedroom hallway (confirm
      fireplace isn't already adjacent to the hallway before buying a 3rd,
      separate unit)
- [ ] Heiman HS1SA-E (smoke): opportunistic only — bedrooms already have smoke
      detectors, replace as each one hits end-of-life rather than a bulk buy

**Not current — barn lighting, 2–3 years out:**

- [ ] Purchase ZWA-2 (Z-Wave) and one Zooz ZEN76/77 800LR switch; range-test at the
      barn's actual switch location before buying more

**Someday / speculative — not a real plan:**

- [ ] Try the Sharp SPC1107 outdoor sensor with an RTL-SDR + rtl_433 (see Legacy
      433MHz Sensors above) — only if looking for an SDR tinkering project
