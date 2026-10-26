# Home Assistant / IoT Network

**Status: early setup.** HA Green is running but minimally configured — no radios added
yet. This doc covers the HA platform itself and device/protocol decisions across the
whole house. Device-specific implementation for the server closet (temp sensor, fan
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

**Decision: run both.** Zigbee (ZBT-2) for the high-volume sensor/plug fleet — same
pattern as the closet's SNZB-02D + smart plug. Z-Wave (ZWA-2) for the door lock and
likely in-wall switches — Z-Wave's certified interop and sub-GHz band (no WiFi
contention, important given how much 2.4GHz traffic this network already has) matter
more than price for security-adjacent and mains-switching hardware.

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
expands. **M5Stack Atom Lite** (~$7.50–10, no soldering, flashes via ESPHome's
browser-based web installer):

- [M5Stack official store](https://shop.m5stack.com/products/atom-lite-esp32-development-kit)
- [Amazon](https://us.amazon.com/M5Stack-Official-Atom-ESP32-Development/dp/B0FB94ZFXY)
- [Pimoroni](https://shop.pimoroni.com/products/atom-lite-esp32-development-kit)
- [The Pi Hut](https://thepihut.com/products/atom-lite-esp32-development-kit)

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

Current: Ultraloq U-Bolt (confirm which variant — WiFi-bridge or Z-Wave). The **Z-Wave
variant has official "Works with Home Assistant" certification** and runs fully local —
no internet dependency for lock/unlock. If the current unit is the WiFi-bridge variant,
it's worth checking whether a Z-Wave module/swap is available, since Z-Wave is the
mature, certified standard for this category generally (Schlage, Yale, Kwikset all
have Z-Wave lines) — this is the main justification for adding the ZWA-2 at all.

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
mains switching — same certified-interop argument as locks, and bundles onto the ZWA-2
already justified by the lock. Zigbee alternatives exist (Third Reality, Sonoff ZBMINI,
IKEA) if keeping everything on one radio is preferred over protocol-matching by
category — worth deciding once the lock situation is settled, since that determines
whether a Z-Wave radio is in place anyway.

## Outstanding / TODO

- [ ] Purchase and set up ZBT-2 (Zigbee) and ZWA-2 (Z-Wave) adapters, USB extensions
      routed out of the closet
- [ ] Confirm which Ultraloq U-Bolt variant is currently installed (WiFi-bridge vs
      Z-Wave) and whether a Z-Wave swap/module is available
- [ ] Pick weather station (Ecowitt vs Tempest) and set up its local HA integration
- [ ] Decide remote light switch protocol (Z-Wave vs Zigbee) once the lock is settled
- [ ] Place first ESPHome Bluetooth proxy (office) for the existing BT lights
- [ ] Pair closet sensor + fan plug once ZBT-2 is up (see
      [11_CLOSET_COOLING.md](11_CLOSET_COOLING.md))
