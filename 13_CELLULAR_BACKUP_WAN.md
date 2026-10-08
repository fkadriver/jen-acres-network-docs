# Cellular Backup WAN (Planned)

Tracking the plan to drop Kinetic/Windstream entirely, promote Starlink to sole
primary WAN, and add a cellular router as pure failover backup.

## Status

**Planning — no hardware purchased yet.** Research done 2026-10-08; see decision
summary below. Depends on [10_STARLINK_MULTIWAN.md](10_STARLINK_MULTIWAN.md), which
is implemented and currently runs Starlink load-balanced 50/50 against Kinetic.

## Goal

- Cancel Kinetic/Windstream (DSL, `em0`, see
  [research/T3200_bridge_mode.md](../research/T3200_bridge_mode.md)).
- Starlink becomes the sole primary WAN (`opt9`/`em2`), not balanced against anything.
- A cellular router provides automatic failover only — idle until Starlink drops,
  not sharing load day-to-day.

## Hardware decision

Same pattern already proven with the Kinetic T3200: put the ISP-side device into
bridge/IP-passthrough mode and hand OPNsense a clean single IP on its own interface.
Cellular backup uses the identical shape — a dedicated cellular router in
passthrough mode, one Ethernet cable into OPNsense on whichever physical port
`em0` vacates once Kinetic is cut. No new ports needed on the Protectli.

| Device | Price | Verdict |
|---|---|---|
| **Teltonika RUTX50** | ~$300–350 | **Leading candidate.** Dual-SIM 5G, clean passthrough-to-WAN mode, widely used for Starlink+cellular failover specifically. |
| Peplink MAX BR1 Mini/Pro 5G, B One 5G | ~$500+ | More polished, adds SpeedFusion bonding (combine rather than just fail over) — likely more than needed for pure backup. |
| GL.iNet Spitz AX (GL-X3000) | ~$170–200 | Cheapest, OpenWrt-based, but forum reports (as of research date) describe bridge/passthrough mode as experimental with no release timeline — risk, not a pick until that matures. |
| USB/mini-PCIe LTE modem in the Protectli itself | varies | Protectli's own backup-WAN guidance targets their Vault line (mini-PCIe slot for an internal modem); the FW41 doesn't have that slot. Not applicable to this hardware. |
| ConnecTen (connecteninternet.com) | $99 hardware + $100/mo unlimited, or $20/day then $11/day | Bundled router+SIM+plan, MiFi/travel-hotspot oriented, not bring-your-own-hardware. No evidence the router supports bridge/IP-passthrough mode, which this architecture requires. Unlimited plan is ~4-5x the cost of the T-Mobile/own-SIM options below. Ruled out unless a passthrough-capable, SIM-only option from them turns up. |

**Decision: Teltonika RUTX50**, pending final purchase.

## Data plan options

| Plan | Cost | Notes |
|---|---|---|
| T-Mobile Home Internet Backup | $20/mo standalone ($10/mo with a T-Mobile voice line) | Dedicated backup plan, 100 hrs/mo uncapped 5G, auto-switching gateway included. Cheapest dedicated option. |
| Own SIM in the RUTX50 (Visible, Tello, Boost, etc.) | ~$25/mo | More carrier flexibility — pick whichever has the best signal at the house; RUTX50's dual-SIM slot also allows a second carrier for redundancy. |

No plan selected yet — depends on signal testing at the house once hardware arrives.

## OPNsense changes required (not yet implemented)

This is a restructuring of the existing Starlink multi-WAN setup
([10_STARLINK_MULTIWAN.md](10_STARLINK_MULTIWAN.md)), not just an addition:

- **Retire `em0` as Kinetic WAN**, repurpose as `CELLULAR` interface once the RUTX50
  is wired in. Check whether the carrier hands back a public or CGNAT address —
  if CGNAT, same "uncheck block private/bogon networks" gotcha documented for
  Starlink's `opt9` may apply.
- **New gateway `CELLULAR_DHCP`** — monitor IP TBD (pick something the carrier
  doesn't filter; Kinetic's ICMP-filtering surprise on `WAN_DHCP` is the
  cautionary example here).
- **Starlink (`STARLINK_DHCP`) moves to Tier 1 alone.** `CELLULAR_DHCP` goes in at
  **Tier 2** — different tiers fail over only, they never share load, per the
  tier semantics already documented in `10_STARLINK_MULTIWAN.md`.
- **Retire the 50/50 `WAN_BALANCE` group**, replace with a Tier1/Tier2 failover
  group (e.g. `STARLINK_FAILOVER`) on the same six client "allow to any" rules
  (LAN, WiFi_Secure, GUEST, SERVERS, HomeAssist, Cailin — see
  [05_FIREWALL_RULES.md](05_FIREWALL_RULES.md)).
- **Revisit the system-default-gateway gap.** `10_STARLINK_MULTIWAN.md` flags that
  `gw_switch_default` is off, so the router's own Tailscale/DNS/NTP traffic doesn't
  fail over — it just goes down with the primary WAN. That gap was accepted while
  Kinetic was still a second physical ISP link; once Starlink is the *only* primary,
  losing it also means losing remote (Tailscale) access to the router unless the
  system default gateway is also pointed at the failover group. Worth fixing as
  part of this change, not deferring again.
- Outbound NAT (Hybrid/`snat_mode`) should auto-cover the new interface the same
  way it did for `em2` — verify, don't assume.
- Sticky connections (`lb_use_sticky`) are already on network-wide; no change needed.

## Open questions / next steps

- [ ] Purchase RUTX50 (or revisit if a better option surfaces before buying).
- [ ] Test cellular signal strength at the house for T-Mobile/Verizon/AT&T before
      picking a data plan or carrier.
- [ ] Cancel Kinetic/Windstream — confirm no contract/ETF penalty first.
- [ ] Wire RUTX50 into freed `em0`, configure passthrough mode.
- [ ] Rebuild OPNsense gateway/group config per above.
- [ ] Decide on `gw_switch_default` fix for router's own traffic.
- [ ] Live failover test (pull Starlink, confirm cellular picks up) — same
      cable-pull verification pattern used for Starlink/Kinetic in
      `10_STARLINK_MULTIWAN.md`.
- [ ] Update `STRUCTURE.md`'s WAN description once implemented.
