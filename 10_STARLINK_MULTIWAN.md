# Starlink WAN + T-Mobile Cellular Backup

**Status: Starlink-only today, updated 2026-10-20.** Kinetic has been turned off.
Starlink (`em2`/`opt9`) is the sole internet connection for now — the 50/50
load-balancing setup documented below (implemented 2026-09-30) was removed the same
day Kinetic went off, since balancing against a dead link made no sense. A T-Mobile
Home Internet Backup cellular connection is **planned** as a Tier2 failover (full plan
in this doc's "T-Mobile Cellular Backup" section below) but isn't cabled/signed up yet.

**The target chain: Starlink → T-Mobile (failover) → OPNsense `em0`.** Starlink stays
primary on `em2`/`opt9`; T-Mobile's gateway is not inline with Starlink (it has no
WAN-in port) — its LAN-out plugs into OPNsense's freed-up `em0` as a second, independent
WAN interface, and OPNsense's own gateway-group logic decides which one carries traffic.

## Current configuration

- **System default gateway**: `STARLINK_DHCP` (flipped from `WAN_DHCP` when load
  balancing was removed — this is what makes Starlink the actual path for any traffic
  not explicitly policy-routed, including the router's own outbound traffic).
- **Client-network rules** (LAN, WiFI_Secure, GUEST, SERVERS, HomeAssist, Cailin): no
  explicit `Gateway` field anymore — removed along with the gateway group, so they
  fall through to the system default (Starlink). See
  [05_FIREWALL_RULES.md](05_FIREWALL_RULES.md).
- **`WAN_BALANCE` gateway group**: deleted. Per the T-Mobile Cellular Backup plan
  below, it won't be recreated in this shape — the next multi-WAN group will be a
  Tier1(Starlink)/Tier2(cellular) *failover* group, not a 50/50 balance.
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

## T-Mobile Cellular Backup (planned, not yet cabled)

Tracking the plan to add T-Mobile Home Internet Backup (cellular) as pure Tier2
failover behind Starlink. Research done 2026-10-08, revised same day after confirming
how T-Mobile's gateway actually works.

### Decision: T-Mobile's own gateway, no third-party router

**Using T-Mobile Home Internet Backup's included gateway directly** — no third-party
cellular router. Scott is already a T-Mobile customer, so this should land at
**$10/mo** (vs. $20/mo standalone), with 100 hrs/mo (130GB) uncapped 5G.

This **replaces an earlier RUTX50 (passthrough router) plan**. Researched and
confirmed:

- **The T-Mobile gateway has no Ethernet WAN-in port — it's LAN-out only**, like a
  basic cellular modem. It cannot take Starlink as an input and fail over on its own;
  it isn't designed to sit "in front of" another connection.
- **It also has no bridge/IP-passthrough mode** — unlike the old Kinetic T3200
  pattern. It's SIM-locked to T-Mobile's own hardware too, so a third-party router
  (RUTX50 or otherwise) can't use the T-Mobile SIM anyway.
- **Conclusion: all failover logic lives in OPNsense**, not in the cellular gateway —
  a Tier1 (Starlink) / Tier2 (cellular) gateway group, with the T-Mobile gateway's LAN
  port plugged into OPNsense's `em0` as a new `CELLULAR` interface instead of a
  passthrough-mode RUTX50.
- **Accept double-NAT** on the cellular path: T-Mobile gateway NATs, then OPNsense
  NATs again behind it. Fine for an idle, outbound-only failover link (no inbound
  hosting over cellular). `em0`/`CELLULAR` will get a private DHCP address from the
  T-Mobile gateway — apply the same "uncheck block private/bogon networks" fix already
  used for Starlink's CGNAT'd `opt9`.
- This also **eliminates the RUTX50 purchase** (~$300–350 saved); the hardware
  comparison below is now moot for this plan and kept only for reference should
  T-Mobile Home Internet Backup fall through (e.g. poor signal at the house).

<details>
<summary>Superseded hardware comparison (own-router + passthrough approach)</summary>

| Device | Price | Verdict |
|---|---|---|
| Teltonika RUTX50 | ~$300–350 | Was leading candidate for a passthrough-mode router approach. No longer needed — T-Mobile's SIM won't work in it anyway. |
| Peplink MAX BR1 Mini/Pro 5G, B One 5G | ~$500+ | More polished, adds SpeedFusion bonding — likely more than needed for pure backup. |
| GL.iNet Spitz AX (GL-X3000) | ~$170–200 | Cheapest, OpenWrt-based, but bridge/passthrough mode reported experimental. |
| USB/mini-PCIe LTE modem in the Protectli itself | varies | Protectli's backup-WAN guidance targets their Vault line; the FW41 doesn't have that slot. |
| ConnecTen (connecteninternet.com) | $99 hardware + $100/mo unlimited, or $20/day then $11/day | Bundled router+SIM+plan, MiFi/travel-hotspot oriented. No evidence of passthrough support; ~4-5x the cost of T-Mobile Home Internet Backup. |
| Own SIM in a third-party router (Visible, Tello, Boost, etc.) | ~$25/mo | More carrier flexibility, but moot without a passthrough-capable router in the mix. |

</details>

Signal strength at the house confirmed acceptable (2026-10-08). **Confirmed directly
by Scott (2026-10-20):** connection runs through T-Mobile's gateway (referred to as the
"4G router"); it only kicks in when Starlink is down — automatic failover only, never
shares load, matching the Tier1/Tier2 plan below; capped at 100 hours/month, matching
the 100 hrs/mo (130GB) figure above.

**Still to confirm:** whether the $10/mo voice-line discount applies automatically to
Scott's existing T-Mobile account/line, or requires a separate sign-up step.

### OPNsense changes required (not yet implemented)

This is a restructuring of the current Starlink-only setup above, not just an
addition:

- **`em0` is freed up (Kinetic turned off 2026-10-20)** — repurpose as `CELLULAR`
  interface once the T-Mobile gateway's LAN port is wired in. It will hand back a
  private DHCP address (double-NAT, confirmed above) — apply the same "uncheck block
  private/bogon networks" gotcha documented above for Starlink's `opt9`.
- **New gateway `CELLULAR_DHCP`** — monitor IP TBD (pick something the carrier doesn't
  filter; Kinetic's ICMP-filtering surprise on the old `WAN_DHCP` is the cautionary
  example here).
- **Starlink (`STARLINK_DHCP`) stays Tier 1 alone.** `CELLULAR_DHCP` goes in at
  **Tier 2** — different tiers fail over only, they never share load, per the tier
  semantics in the "Prior setup" section above.
- **New Tier1/Tier2 failover group** (e.g. `STARLINK_FAILOVER`) on the same six client
  "allow to any" rules (LAN, WiFi_Secure, GUEST, SERVERS, HomeAssist, Cailin — see
  [05_FIREWALL_RULES.md](05_FIREWALL_RULES.md)) — not a 50/50 balance like the old
  `WAN_BALANCE` group.
- **Revisit the system-default-gateway gap.** That gap (no `gw_switch_default`) was
  accepted while Kinetic was still a second physical ISP link; once there's a real
  Tier2 failover candidate again, losing Starlink should also fail over the router's
  own Tailscale/DNS/NTP traffic, not just client traffic — worth fixing as part of this
  change.
- Outbound NAT (Hybrid/`snat_mode`) should auto-cover the new interface the same way it
  did for `em2` — verify, don't assume.
- Sticky connections (`lb_use_sticky`) are already on network-wide; no change needed.

### Open questions / next steps

- [x] Test T-Mobile cellular signal strength at the house — confirmed acceptable
      (2026-10-08).
- [ ] Sign up for T-Mobile Home Internet Backup; confirm $10/mo (not $20/mo) applies
      to Scott's existing account/line.
- [x] Kinetic/Windstream turned off (2026-10-20).
- [ ] Wire T-Mobile gateway's LAN port into freed `em0` (no passthrough config — it's
      a standalone double-NAT device; see decision above).
- [ ] Rebuild OPNsense gateway/group config per above.
- [ ] Decide on `gw_switch_default` fix for router's own traffic.
- [ ] Live failover test (pull Starlink, confirm cellular picks up) — same cable-pull
      verification pattern used for Starlink/Kinetic above.
- [ ] Update `STRUCTURE.md`'s WAN description once implemented.

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
