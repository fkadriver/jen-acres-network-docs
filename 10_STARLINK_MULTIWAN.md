# Starlink Multi-WAN (Load Balanced)

**Status: implemented and verified (2026-09-30).** Starlink V5 (100Mbps) runs as a
second WAN on `em2`/`opt9`, load-balanced 50/50 against the existing primary WAN
("Kinetic", `em0`) since both connections are the same speed. Both gateways have
active health monitoring and have been verified via two live cable-pull tests.

## Prerequisites

- Starlink router set to **Bypass mode** (IP Passthrough) via the Starlink app before
  wiring anything to OPNsense. Without this you get double-NAT and OPNsense won't see
  a real WAN-facing address. (Confirmed live: without Bypass, `em2` picked up a private
  `192.168.1.x` lease from the Starlink router's own DHCP; with Bypass on, it correctly
  gets a CGNAT address.)
- Expect a **CGNAT address (100.64.0.0/10)** from Starlink even in Bypass mode — this
  is normal, not a misconfiguration.
- `em2` was the free physical NIC (see [01_OPNSENSE_INSTALLATION.md](01_OPNSENSE_INSTALLATION.md)) —
  no hardware changes needed, just cabling. It's assigned as interface `opt9`,
  described `STARLINK`.

## Interface (`opt9` / `em2`)

| Setting | Value |
|---|---|
| Description | `STARLINK` |
| IPv4 Configuration Type | DHCP |
| Block private networks | off |
| Block bogon networks | off |

Unchecking both blocks is required — Starlink's CGNAT range (100.64.0.0/10) reads as
private/bogon space to OPNsense and the interface will silently lose connectivity
otherwise. This is the most common Starlink+OPNsense gotcha.

## Gateways

| Gateway | Interface | Monitor IP | Weight | Why |
|---|---|---|---|---|
| `STARLINK_DHCP` | opt9 | `1.1.1.1` | 1 | Responds cleanly via Starlink (0% loss, ~20ms) |
| `WAN_DHCP` | wan (em0) | `192.168.254.254` (Kinetic's own modem) | 1 | **Kinetic filters outbound ICMP to internet destinations** — both `1.1.1.1` and `8.8.8.8` showed sustained 100% loss via em0 despite real HTTPS traffic working fine. Monitoring the ISP's own directly-connected gateway is the standard fix when an ISP blocks ICMP to public targets. This only detects "link to the modem is down," not a deeper upstream Kinetic outage past the modem — that's the best available signal without a TCP/HTTP probe mode, which OPNsense's built-in gateway monitor doesn't have. |

Equal weight on both = a straight 50/50 split, matching the equal 100Mbps link speeds.

## Gateway Group: `WAN_BALANCE`

Both gateways at **Tier 1** (equal tier = load-balance/shared; different tiers would
only fail over, never share). Trigger: `down` — a member is pulled from rotation when
its monitor alarms down, and returned once it alarms back up.

## Outbound NAT

Mode is **Hybrid** (`snat_mode` in the new Firewall/Filter model), which auto-generates
outbound NAT for any interface with a gateway — confirmed `em2`/STARLINK is covered for
every internal subnet without any manual NAT rule needed.

## Policy Routing (which traffic is balanced)

The gateway group only affects traffic whose firewall rule explicitly sets
**Gateway = `WAN_BALANCE`**. Six client-network "allow to any" rules carry this:

- LAN (MGMT_LAN)
- WiFI_Secure (opt1)
- GUEST (opt2)
- SERVERS (opt3)
- HomeAssist (opt5)
- Cailin (opt7)

See [05_FIREWALL_RULES.md](05_FIREWALL_RULES.md) for the per-interface rule context.

**Deliberately excluded from the balance** (stay on the system default gateway,
`WAN_DHCP`):

- Tailscale (opt4) and all Tailscale-sourced/management traffic — mid-session gateway
  changes can break `tailscaled`'s established connections.
- DMZ (opt8) — physically bypasses the router straight to the modem, never touches
  these gateways at all.
- The router's own outbound traffic (Tailscale daemon, DNS, NTP, package updates) —
  there is no automatic default-gateway failover configured (`gw_switch_default` is
  off), so a primary-WAN outage will drop the router's own Tailscale/SSH reachability
  even though client traffic fails over cleanly. This is a known, accepted gap — fixing
  it would require setting the *system* default gateway to a group, which has broader
  implications and hasn't been done.

## Sticky Connections

Already enabled network-wide (`lb_use_sticky = 1`, System → Settings → Advanced →
Firewall/NAT) before this change — no extra step needed. Compiled pf rule confirms it:
`route-to { (em0 ...), (em2 ...) } round-robin sticky-address`. This pins an
already-established session (by source IP) to whichever gateway it first landed on,
rather than letting the pool re-roll into a different gateway on new states from the
same source — important for VoIP, gaming, and anything long-lived.

## Verification (completed 2026-09-30)

- [x] `opt9`/`em2` up with a real Starlink CGNAT lease (`100.x.x.x/10`)
- [x] Both gateways online with real monitor stats (not `~` placeholders)
- [x] `WAN_BALANCE` compiled into pf as `route-to {...} round-robin sticky-address`
      on all six client rules
- [x] Outbound NAT covers `em2` for every internal subnet
- [x] **Live cable-pull test #1** (Kinetic): exposed that `WAN_DHCP` had no active
      monitor at all — the group couldn't have detected a real Kinetic outage.
- [x] **Live cable-pull test #2** (after fixing the monitor): confirmed in
      `/var/log/gateways/latest.log` — `WAN_DHCP` alarmed `none -> loss -> down`
      within ~16s of the cable coming out, and `down -> none` immediately on
      reconnect.

## Known limitations

- No system-default-gateway failover — see "Deliberately excluded" above. A real
  Kinetic outage drops remote (Tailscale) management access to the router itself,
  even though household client internet correctly shifts to Starlink-only.
- `WAN_DHCP`'s health check only reaches Kinetic's own modem, not the broader
  internet — an outage upstream of the modem (but with the modem itself still up)
  would not be detected, and the balance would keep trying to send ~50% of new
  connections through a gateway that's actually not working.
- The 50/50 split is a trial/observation state, not necessarily permanent — Starlink
  may become the primary or only WAN once its reliability is confirmed over time, at
  which point the weights/tiers here should be revisited rather than assumed fixed.
