# Server Closet Cooling (HVAC Return Tap + HA-Automated Fan)

**Status: planning — not yet implemented.** Site survey of the closet ceiling is done
(see below). **Next blocking step: get a dedicated 20A circuit/outlet run to the
closet** before mounting the fan or anything else permanent.

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

**Decision: a wall transfer vent into the bedroom, not a door modification.** Cut low
on the wall behind nas01 (bottom-left of the closet), paired with the return grille at
the opposite top corner (upper-right, near the ceiling duct bay). Two reasons this
beats the door options:

- **Better airflow path.** Diagonal, low-to-high, source-to-exhaust — makeup air enters
  right where the NAS (the biggest heat source) sits and has to rise/sweep across the
  full diagonal of the closet to reach the exit, riding the same direction as natural
  convection instead of crossing it. The door-level options would only sweep width-wise
  near the floor.
- **Easier to revert.** A drywall patch if the server closet ever moves is simpler than
  restoring a modified or swapped door.

Because this vent opens directly into the bedroom, use a **sound-baffled/offset
transfer vent kit** (insulated duct box between the two grilles, not a straight
grille-on-grille hole) — cuts down on fan and equipment noise reaching the bedroom at
night, not just airflow resistance.

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

- **Intake**: wall transfer vent, low, behind nas01 (bottom-left)
- **Exhaust**: ceiling return grille, far end, near the equipment rack (upper-right)

This pulls makeup air across and up through the full equipment run — both the 8' length
and the vertical rise of the rack — before it exits into the return, rather than
short-circuiting near one end.

## Parts List

### Makeup Air / Wall Transfer Vent

- A **sound-baffled wall transfer vent kit** (e.g. Home Intuition or Imperial
  sound-dampening transfer vent, ~$30–50) — insulated/lined duct box between two
  grilles rather than a straight hole, to cut down on fan/equipment noise reaching the
  bedroom. Sized to the stud bay (typically 3.5" deep for a 2x4 wall).
- Cut low, behind nas01, on the closet side; terminates on the bedroom side at a
  matching low wall location.

### Fan

Dedicated inline duct fan, run at fixed speed off a plain switched outlet — Home
Assistant handles the on/off thermostat logic, not the fan's own controller.

- **AC Infinity CLOUDLINE S6** (6", ~165 CFM, ~$55) — quiet at low speed, sized with
  headroom over the actual load (~100–150 CFM needed). Buy the standalone fan, not the
  kit with AC Infinity's UIS speed controller.
- Budget alternative: iPower or TerraBloom 4" inline duct fan (~$30) — basic fixed-speed
  AC fan, no controller required.

### Power

- **Fan**: Zigbee smart plug, ideally energy-monitoring (confirms the fan is actually
  drawing current — a stalled motor shows up as 0W instead of failing silently).
  SONOFF S40 Lite (Zigbee) or Third Reality Smart Plug, ~$12–15.
- **Switch / OPNsense box**: fine to put on a switchable Zigbee plug for remote
  power-cycle capability — low risk.
- **NAS — do not put on a switchable smart plug.** Accidental automation or dashboard
  mis-tap cutting power risks filesystem corruption. Prefer the native Synology/QNAP
  integration for wattage/health visibility, or a monitor-only (non-switching)
  energy-sensing plug if the NAS doesn't expose it natively.
- **UPS — not a smart plug either.** Use Network UPS Tools (NUT) — HA add-on or a small
  box talking to the UPS over USB — for battery %, load wattage, runtime, and a clean
  "on battery" signal/shutdown trigger.

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

- **Home Assistant Connect ZBT-1** (official, ~$35), on a 3–6 ft USB extension cable,
  mounted away from the closet's metal shelving/rack and WiFi APs — Zigbee shares the
  2.4GHz band, and that closet is a rough RF environment for it.
- Cheaper alternative: Sonoff ZBDongle-E (~$20, same Silicon Labs chip).
- Favor Sonoff/Third Reality Zigbee devices over Aqara initially — Aqara has had known
  pairing quirks on a ZHA mesh with few routers present.

## Outstanding / TODO

- [ ] **Dedicated 20A circuit/outlet in the closet** (blocking — needed before
      mounting the fan, and ideally sized for the existing rack load too)
- [ ] **Cut wall transfer vent behind nas01** (bottom-left, into bedroom) — required
      for the return tap to work at all; door is solid, so this is the makeup-air path
- [ ] Identify the second flex duct in the joist bay before cutting near it
- [ ] Cut return grille opening, clear local insulation, seal drywall-to-joist edge
- [ ] Mount fan + Zigbee plug, wire to the new 20A circuit
- [ ] Pair Zigbee coordinator, set up ZHA, pair sensor + plug
- [ ] Configure the Generic Thermostat helper in HA
- [ ] Validate: confirm closet temp tracks near room ambient under normal load

## References

- Site photos: `../Closet/` (IMG_4496–4498, HEIC — not yet converted/committed as
  viewable images)
