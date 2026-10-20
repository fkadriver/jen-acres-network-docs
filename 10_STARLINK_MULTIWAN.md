# Starlink WAN (Sole Connection)

**Status: Starlink-only, updated 2026-10-20.** Kinetic has been turned off. A T-Mobile
Home Internet Backup cellular connection is planned as a Tier2 failover (see
[13_CELLULAR_BACKUP_WAN.md](13_CELLULAR_BACKUP_WAN.md)) but isn't cabled/signed up
yet. Starlink (`em2`/`opt9`) is the sole internet connection for now — the 50/50
load-balancing setup documented below (implemented 2026-09-30) was removed the same
day Kinetic went off, since balancing against a dead link made no sense.

## Current configuration

- **System default gateway**: `STARLINK_DHCP` (flipped from `WAN_DHCP` when load
  balancing was removed — this is what makes Starlink the actual path for any traffic
  not explicitly policy-routed, including the router's own outbound traffic).
- **Client-network rules** (LAN, WiFI_Secure, GUEST, SERVERS, HomeAssist, Cailin): no
  explicit `Gateway` field anymore — removed along with the gateway group, so they
  fall through to the system default (Starlink). See
  [05_FIREWALL_RULES.md](05_FIREWALL_RULES.md).
- **`WAN_BALANCE` gateway group**: deleted. Per the plan in
  [13_CELLULAR_BACKUP_WAN.md](13_CELLULAR_BACKUP_WAN.md), it won't be recreated in
  this shape — the next multi-WAN group will be a Tier1(Starlink)/Tier2(cellular)
  *failover* group, not a 50/50 balance.
- **Kinetic (`WAN_DHCP`, `em0`)**: gateway definition and interface config currently
  left in place, unused (`defaultgw` off). Per the cellular backup plan, `em0` is
  slated to be **repurposed** as the `CELLULAR` interface (T-Mobile gateway's LAN
  port wired in) rather than kept around for Kinetic, which isn't expected to come
  back. `WAN_DHCP`'s old monitor target (Kinetic's own modem, `192.168.254.254` — see
  "Prior setup" for why a public IP didn't work there) is moot once that happens.

## Starlink interface (`opt9` / `em2`)

| Setting | Value |
|---|---|
| Description | `STARLINK` |
| IPv4 Configuration Type | DHCP |
| Block private networks | off |
| Block bogon networks | off |

Unchecking both blocks is required — Starlink's CGNAT range (100.64.0.0/10) reads as
private/bogon space to OPNsense and the interface will silently lose connectivity
otherwise. This is the most common Starlink+OPNsense gotcha. Expect a CGNAT address
(`100.x.x.x/10`), not a public IP, even in Bypass mode — normal, not a misconfiguration.

Starlink router must be in **Bypass mode** (IP Passthrough) for this to work —
without it you get double-NAT and OPNsense sees a private `192.168.1.x`-style lease
from the Starlink router's own DHCP instead of a CGNAT address.

## T-Mobile backup (planned, not yet cabled)

Full plan and research: [13_CELLULAR_BACKUP_WAN.md](13_CELLULAR_BACKUP_WAN.md).
Short version: T-Mobile Home Internet Backup's gateway has **no bridge/passthrough
mode** — it's a standalone double-NAT device, LAN-out only. It isn't sitting
physically "inline" between Starlink and OPNsense; its LAN port wires into a
repurposed `em0` (freed up now that Kinetic is off) as a new `CELLULAR` interface,
and **all failover logic lives in OPNsense** as a Tier1 (Starlink) / Tier2
(cellular) gateway group — failover only, never sharing load, unlike the old 50/50
`WAN_BALANCE` setup. Not yet implemented; nothing purchased/signed up yet as of
2026-10-08.

## Prior setup: Starlink + Kinetic load balancing (2026-09-30 to 2026-10-20)

For reference, in case multi-WAN comes back in a similar shape — this is what was
implemented, verified, and then removed:

- Two gateways, `STARLINK_DHCP` (monitor `1.1.1.1`) and `WAN_DHCP` (monitor
  `192.168.254.254`, Kinetic's own modem — **Kinetic filters outbound ICMP to public
  destinations**, so a public monitor IP showed sustained 100% loss despite real
  HTTPS traffic working fine; monitoring the ISP's own directly-connected gateway is
  the standard fix for that), both weight 1 for an even 50/50 split.
- A `WAN_BALANCE` gateway group, both members at Tier 1 (equal tier = load-balance;
  different tiers would only fail over).
- The same 6 client-network rules had `Gateway = WAN_BALANCE` set, compiling into pf
  as `route-to { (em0 ...), (em2 ...) } round-robin sticky-address`.
- Tailscale/management traffic and DMZ were deliberately kept off the balanced group.
- Verified via two live cable-pull tests (the first of which exposed that `WAN_DHCP`
  had no active monitor at all, fixed before the second confirmed correct
  down/recovery detection in `/var/log/gateways/latest.log`).
- Known limitation at the time: no system-default-gateway failover was configured, so
  a Kinetic outage would've dropped the router's own Tailscale/SSH reachability even
  though client traffic correctly shifted to Starlink-only. Moot now that Starlink is
  the system default outright.
