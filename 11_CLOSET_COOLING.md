# Server Closet Cooling (HVAC Return Tap + HA-Automated Fan)

**Status: live and running, 2026-11-04.** Physical install (vent/grille cutting, fan
mounting) is done; Generic Thermostat helper is configured and confirmed actively
controlling the fan. Only remaining item is routing the Zigbee USB extension out to
the hallway (cosmetic/RF-hygiene, not functional) and longer-run validation.

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

**HA Green** sits on top of the metal cabinet on the left — not enclosed in metal, but
still worth a USB extension cable for the Zigbee adapter, routed out of the closet
entirely and into the hallway (same opening as the makeup-air vent) rather than left
sitting in the closet's RF-noisy environment. Radio hardware details are in
[13_HOME_ASSISTANT.md](13_HOME_ASSISTANT.md).

## Parts List

### Makeup Air / Wall Transfer Vent

- A plain **wall transfer grille/vent kit** (~$10–20 for a pair of grilles, no sound
  baffling needed since it opens into the hallway, not the bedroom). Sized to the stud
  bay (typically 3.5" deep for a 2x4 wall).
- Cut low, behind nas01, on the closet side; terminates on the hallway side at a
  matching low wall location.
- **Filter: cut-to-fit MERV 8 filter media roll** — trims down to as small as 4"×10"
  or 4"×12", smaller than any rigid pre-sized filter, so it's cut to the grille's
  actual opening rather than hunting for a matching pre-made size. MERV 8 specifically
  — catches normal household dust without the pressure drop higher MERV ratings would
  add, which matters given the design's limited CFM headroom (~95 needed, ~100 rated).
  Stick the patch to the inside of the closet-side grille (most rolls ship with
  adhesive/hook tape for this) and keep it easily removable — it's pulling air
  continuously whenever the fan runs, so it'll need periodic cleaning/replacement.

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

**Fan control: SONOFF S40 Lite (Zigbee smart plug), ~$15 — purchased and paired.** The
fan (AC Infinity CLOUDLINE S4 or the TerraBloom/iPower alternative) ships with a
standard power cord — no hardwiring or inline relay needed, just plug the fan into the
S40 Lite and the S40 Lite into a wall outlet. Picked over a generic plug for two
reasons:

- **Acts as a Zigbee router**, extending mesh coverage from right at the far end of the
  closet — useful given the RF concerns already noted for this space (metal shelving,
  WiFi APs).
- Pairs natively with ZHA, no cloud dependency, consistent with everything else in
  this plan.

**Correction: no energy monitoring.** Originally picked partly for built-in energy
monitoring, but confirmed after pairing — ZHA only exposes `switch`, firmware `update`,
`identify`, and radio diagnostics (`lqi`/`rssi`) for this device; no watts/kWh sensor.
The Zigbee S40 Lite doesn't have a metering cluster (that's the WiFi S40, a different
product). A stalled fan motor won't show up as 0W — there's no power entity to check.
If current-sensing matters enough to revisit, a plug with a metering cluster (e.g. SONOFF
S60ZBTPF or similar) would be the fix; not planned for now since the switch/router
function is what's actually needed here.

This is the switch entity the Generic Thermostat helper below controls — entity id
`switch.network_closet_closet_fan`. Still need the fan itself mounted — the plug is
paired but has nothing to switch yet.

- NAS and UPS: not on switchable smart plugs (accidental cutoff risks filesystem
  corruption) — NAS via native Synology/QNAP integration, UPS via NUT.
- Switch/OPNsense box: fine on a switchable Zigbee plug too, if wanted, using the same
  pick.

### Sensor / "Thermostat"

- **SONOFF SNZB-02D** Zigbee temp/humidity sensor (~$15) — purchased and paired as
  `SONOFF SNZB-02DR2`. Mount **near the fan/return tap, not inside the metal rack** —
  the rack would both shield the Zigbee signal and read rack-internal temp rather than
  the air actually leaving the closet. Relevant entities:
  `sensor.closet_temp_temperature`, `sensor.closet_temp_humidity`,
  `sensor.closet_temp_battery`.
- **Configured and live** as the built-in **Generic Thermostat** helper (Settings →
  Devices & Services → Helpers → *Generic Thermostat*), entity
  `climate.network_closet_closet_fan_network_closet_temp`:
  - Heater (switch) entity: `switch.network_closet_closet_fan`
  - Temperature sensor: `sensor.closet_temp_temperature`
  - **AC mode: ON** — confirmed correct; this is a cooling setup (fan runs when too
    hot), and Generic Thermostat defaults to heater logic without it.
  - Target temp: **70°F**, hot/cold tolerance ±2°F — tune after living with it.
  - Mode: **Cool** (active). Verified end-to-end 2026-11-04: closet at 76°F,
    `hvac_action: cooling`, fan switch `on`.
- **Two automations on the same 90°F/5-min trigger, kept separate on purpose** (one
  disables independently of the other):
  - `closet_high_temp_alert` — mobile notification via `notify.iphone`, early warning
    for a dead fan or blocked return.
  - `closet_high_temp_shutdown` — gracefully shuts down **nas01, vm01, log01,
    sands-bak01** (not the router/switch/APs, so the network itself stays up). Each
    host runs a small Tailscale-only authenticated HTTP listener
    (`shutdown-listener`, NixOS module for nas01/vm01/log01, hand-installed via
    `hosts/sands-bak01/setup.sh` step 17 since that host isn't NixOS); HA calls it
    via `rest_command` rather than SSH, since HA Core's container has no SSH client
    and the servers sit on a different VLAN — same "route it over Tailscale instead
    of punching a firewall hole" pattern used for the NAS backup NFS export (see
    [12_UPS_POWER.md](12_UPS_POWER.md)).

**Closet temp also mirrored to each server locally** — a third automation,
`closet_temp_status_push`, fires on every `sensor.closet_temp_temperature` update and
POSTs the current reading + a 90°F-above flag to each host's `shutdown-listener` at
`/status`. Run **`closet-status`** directly on nas01/vm01/log01/sands-bak01 to see it
(e.g. `Closet temp: 78.0F (updated 42s ago)`) — no HA round-trip needed, useful for
checking "why did this reboot" from the host itself after a shutdown.

## Outstanding / TODO

Software side (Zigbee pairing, channel, helper config) is done or spec'd. Physical
side is next — ordered below by actual dependency, top to bottom:

1. [x] **Identify the second flex duct in the joist bay** — done
2. [x] **Cut return grille opening** (ceiling, upper-right, near equipment rack), clear
       local insulation, seal drywall-to-joist edge — done; this is also where the fan
       physically mounts (step 4)
3. [x] **Cut wall transfer vent behind nas01** (bottom-left, into hallway) — the
       makeup-air path — done
4. [x] **Mount fan + Zigbee plug, plug into power** — installed
5. [ ] **Route Zigbee adapter's USB extension out to the hallway** (radio setup itself
       is tracked in [13_HOME_ASSISTANT.md](13_HOME_ASSISTANT.md)) — same opening as
       the makeup-air vent, bundle with step 3's work
6. [x] Set up closet high-temp alert automation — notifies if
       `sensor.closet_temp_temperature` exceeds 90°F for 5+ min, via `notify.iphone`.
       Doesn't depend on the fan being mounted, so done now
6b. [~] Set up closet high-temp **shutdown** automation (nas01/vm01/log01/
       sands-bak01) plus the `closet_temp_status_push` status mirror — same
       trigger, separate automations. HA-side config written (`rest_command`
       x8 + both automations), done via restarting HA Core. NixOS-side
       `shutdown_listener_token` sops secret still needs to be set
       (`sops --set` command provided separately) and nas01/vm01/log01
       rebuilt; sands-bak01 needs the token written by hand and the listener
       service started (setup.sh manual step 6) — until then the shutdown
       and `closet-status` pieces are wired on the HA side but not yet live
       on any of the four hosts
7. [x] Configure the Generic Thermostat helper in HA — done. Setpoint raised
       70°F → **75°F**; min/max range widened to 60–85°F (initial setup had
       left `max_temp` capped at 70, which silently capped the UI slider —
       fixed by editing `core.config_entries` directly and reloading). AC
       mode on. Fan plug dropped off the Zigbee mesh right after mounting
       (blinking green, no last_seen activity even after a power cycle) —
       re-paired via ZHA's Add Device flow, came back clean
8. [~] Validate: confirmed the control loop works end-to-end (76°F closet temp →
       `hvac_action: cooling` → fan switch `on`), 2026-11-04. Longer-run validation
       (does it actually track down toward room ambient over time) still pending —
       check back after it's had a chance to run

Already done:

- [x] Pair fan plug to the Zigbee mesh — SONOFF S40 Lite paired as
      `switch.network_closet_closet_fan` (coordinator: SONOFF ZBDongle-E)
- [x] Pair temp/humidity sensor — SONOFF SNZB-02DR2 paired, entities include
      `sensor.closet_temp_temperature` and `sensor.closet_temp_humidity`
- [x] Lock Zigbee network to channel 25 (migrated off the WiFi-overlapping channel 11
      ZHA auto-picked at setup)

## References

- Site photos: `../Closet/` (IMG_4496–4498, HEIC — not yet converted/committed as
  viewable images)
