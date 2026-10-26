# Server Closet Cooling (HVAC Return Tap + HA-Automated Fan)

**Status: planning — not yet implemented.** Site survey of the closet ceiling is done
(see below). Electrical prerequisite (dedicated 20A circuit) is tracked in
[12_UPS_POWER.md](12_UPS_POWER.md).

## Problem

Network/server gear (nas01, switch, OPNsense router, UPS, misc) all lives in a 2'×8'
bedroom closet with a solid door and no dedicated cooling. Heat load is modest
(~150–450W continuous, roughly 500–1500 BTU/hr) but with no path out, it builds up in
the closet over the ambient bedroom temperature. Goal: remove that heat continuously
without dumping cold air into the closet (which would overcool the attached bedroom)
and without a dedicated AC/mini-split, which is oversized and impractical for an
interior closet with no exterior wall.

## Design

**Tap the HVAC return, not the supply.** A supply tap only helps while the compressor
is actively cooling; a return tap pulls closet air out any time air is moving (even
fan-only mode), and spreads the heat into the whole house's air mass instead of
concentrating cold air in one small room. This requires a makeup-air path into the
closet to work — as the return tap depressurizes the closet slightly, replacement air
needs somewhere to come in from the room.

**Door is solid, not louvered — a makeup-air path has to be added.** With a sealed
door, the fan would be pulling against a closed box: starved airflow at best, or
straining against the door/frame seal at worst.

**Decision: a wall transfer vent into the adjacent hallway, not the bedroom, and not a
door modification.** Cut low on the wall behind nas01 (bottom-left of the closet, on
the hallway side), paired with the return grille at the opposite top corner
(upper-right, near the ceiling duct bay). Reasons this beats the alternatives:

- **Better airflow path.** Diagonal, low-to-high, source-to-exhaust — makeup air enters
  right where the NAS (the biggest heat source) sits and has to rise/sweep across the
  full diagonal of the closet to reach the exit, riding the same direction as natural
  convection instead of crossing it. The door-level options would only sweep width-wise
  near the floor.
- **Easier to revert.** A drywall patch if the server closet ever moves is simpler than
  restoring a modified or swapped door.
- **Hallway over bedroom.** The hallway runs cooler than the bedroom (not held at
  sleeping-comfort setpoint), so makeup air is pulling in air that's already closer to
  ambient. It also means no one's trying to sleep next to the vent, so **no sound
  baffling is required** — a plain transfer grille is fine, simpler and cheaper than the
  sound-baffled kit a bedroom-facing vent would have needed.

Chosen over a standalone AC/mini-split because the actual heat load is small — a
dedicated unit's minimum output is almost always higher than needed, causing
short-cycling, and there's no exterior wall or window for a line-set/condensate drain
anyway.

## Site Survey

Closet ceiling drywall was already open for inspection (photos: `../Closet/`,
IMG_4496–4498, HEIC). Findings:

- **The closet is fully interior, but the ceiling opens into a joist bay above that is
  the home's return path** — confirmed joist-to-joist, i.e. a panned-joist return
  (the framing cavity itself serves as return duct back to the air handler). This means
  **no new ductwork is needed** — just a return air grille cut directly into the
  closet ceiling drywall, seated into that open bay with a short boot.
- A large insulated round duct crosses that same bay — this is a **supply duct just
  passing through**, not the return path. Leave it alone; cut the grille opening clear
  of it.
- Fiberglass batt insulation is stuffed into the bay, likely restricting airflow near
  the tap point — needs to be pulled/trimmed back locally where the grille is cut.
- 12-gauge Romex (yellow jacket) runs through a drilled joist hole in the same bay —
  keep the cut and any fasteners clear of it.
- A second, smaller flex duct drops through a separate joist hole nearby — purpose not
  yet confirmed. Identify it before cutting anything close to it.
- Panned-joist returns are inherently a bit leaky/dusty by construction (pulling air
  from wall cavities / rim joist gaps) — not something this project introduces, but
  worth sealing the new grille's drywall-to-joist edge with mastic or foil tape so it's
  not an extra uncontrolled leak on top of what's already there.

## Placement

Diagonal flow across the closet, low to high:

- **Intake**: wall transfer vent, low, behind nas01 (bottom-left), into the hallway
- **Exhaust**: ceiling return grille, far end, near the equipment rack (upper-right)

This pulls makeup air across and up through the full equipment run — both the 8' length
and the vertical rise of the rack — before it exits into the return, rather than
short-circuiting near one end.

## Parts List

### Makeup Air / Wall Transfer Vent

- A plain **wall transfer grille/vent kit** (~$10–20 for a pair of grilles, no sound
  baffling needed since it opens into the hallway, not the bedroom). Sized to the stud
  bay (typically 3.5" deep for a 2x4 wall).
- Cut low, behind nas01, on the closet side; terminates on the hallway side at a
  matching low wall location.

### Fan

Dedicated inline duct fan, run at fixed speed off a plain switched outlet — Home
Assistant handles the on/off thermostat logic, not the fan's own controller (so there's
no "runs quieter at low speed" benefit to oversizing — it's full speed whenever it's
on).

**4", not 6".** CFM math: at the high end of the heat estimate (~450W / 1536 BTU/hr)
and a 15°F rise, required airflow is ~95 CFM. A 4" fan rated ~100 CFM free-air covers
that with margin, since this is a short, low-resistance tap rather than a long duct
run. 6" buys no real benefit here and means a bigger, less necessary hole in the flat
sheet metal pan at the return tap.

- **AC Infinity CLOUDLINE S4** (4", ~100 CFM, ~$45) — buy the standalone fan, not the
  kit with AC Infinity's UIS speed controller.
- Budget alternative: iPower or TerraBloom 4" inline duct fan (~$30) — basic fixed-speed
  AC fan, no controller required.
- **Mounting**: the return tap lands on a flat sheet-metal pan, not round duct — cut a
  round hole sized to a standard 4" sheet-metal starting collar/boot, screw the collar
  flange to the pan, seal the edge (mastic/foil tape), then mount the fan to the
  collar.

### Power

Smart plugs/relays already on hand — **just need the physical 20A outlet installed**
(see [12_UPS_POWER.md](12_UPS_POWER.md)).

- NAS and UPS: not on switchable smart plugs (accidental cutoff risks filesystem
  corruption) — NAS via native Synology/QNAP integration, UPS via NUT.
- Fan and switch/OPNsense box: fine on switchable Zigbee plugs.

### Sensor / "Thermostat"

- **SONOFF SNZB-02D** Zigbee temp/humidity sensor (~$15), mounted near the top of the
  closet close to the new return grille where heat concentrates.
- In Home Assistant, use the built-in **Generic Thermostat** helper (Settings → Devices
  & Services → Helpers → *Generic Thermostat*) rather than hand-written automation
  YAML: sensor = SNZB-02D, switch = fan's Zigbee plug, set target temp + hysteresis.
  Produces a native `climate` entity with a dashboard thermostat card.
- Optional second automation: mobile notification if closet temp exceeds a critical
  threshold (e.g. 90°F) — early warning for a dead fan or blocked return.

### Zigbee Coordinator (for HA Green)

HA Green has no built-in radio.

- **Home Assistant Connect ZBT-2** (official successor to the ZBT-1, which is now
  discontinued, ~$49) — Zigbee/Thread/Matter USB adapter, better antenna and faster
  UART than the ZBT-1. On a 3–6 ft USB extension cable, mounted away from the closet's
  metal shelving/rack and WiFi APs — Zigbee shares the 2.4GHz band, and that closet is
  a rough RF environment for it. [Official product page](https://www.home-assistant.io/connect/zbt-2/),
  also at [Ameridroid](https://ameridroid.com/products/home-assistant-connect-zbt-2)
  and [Everything Smart](https://shop.everythingsmart.io/products/home-assistant-connect-zbt-2).
- Cheaper alternative: **Sonoff ZBDongle-E** (~$18–30, same Silicon Labs EFR32MG21
  chip) — [Amazon](https://www.amazon.com/sonoff-zigbee-dongle/s?k=sonoff+zigbee+dongle),
  [AliExpress](https://www.aliexpress.us/item/3256803932977988.html).
- Favor Sonoff/Third Reality Zigbee devices over Aqara initially — Aqara has had known
  pairing quirks on a ZHA mesh with few routers present.

### Bluetooth (for cheap BLE LED strips)

Generic BLE LED strips (no WiFi, straight Bluetooth) need real local BT coverage where
they actually are — not a single adapter stuck in the closet, which has the same bad
2.4GHz RF environment as the Zigbee radio and is probably nowhere near the strips
anyway.

- **ESPHome Bluetooth proxy** — a cheap ESP32 board flashed with ESPHome's proxy
  firmware, placed in whichever room(s) have the strips, extending BLE range to HA over
  WiFi instead. **M5Stack Atom Lite** (~$7.50–10, no soldering, flashes via ESPHome's
  browser-based web installer) is the standard pick:
  [M5Stack official store](https://shop.m5stack.com/products/atom-lite-esp32-development-kit),
  [Amazon](https://us.amazon.com/M5Stack-Official-Atom-ESP32-Development/dp/B0FB94ZFXY),
  [Pimoroni](https://shop.pimoroni.com/products/atom-lite-esp32-development-kit),
  [The Pi Hut](https://thepihut.com/products/atom-lite-esp32-development-kit).
- Budget one per room/area with actual BLE devices rather than trying to cover the
  whole house from a single proxy.

## Outstanding / TODO

- [ ] **Cut wall transfer vent behind nas01** (bottom-left, into hallway) — required
      for the return tap to work at all; door is solid, so this is the makeup-air path
- [ ] Identify the second flex duct in the joist bay before cutting near it
- [ ] Cut return grille opening, clear local insulation, seal drywall-to-joist edge
- [ ] Mount fan + Zigbee plug, wire to the new 20A circuit (see
      [12_UPS_POWER.md](12_UPS_POWER.md))
- [ ] Pair Zigbee coordinator, set up ZHA, pair sensor + plug
- [ ] Flash and place ESPHome Bluetooth proxy(s) for the BLE LED strips
- [ ] Configure the Generic Thermostat helper in HA
- [ ] Validate: confirm closet temp tracks near room ambient under normal load

## References

- Site photos: `../Closet/` (IMG_4496–4498, HEIC — not yet converted/committed as
  viewable images)
