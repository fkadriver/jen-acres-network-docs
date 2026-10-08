# Cellular Backup WAN (Planned)

Tracking the plan to drop Kinetic/Windstream entirely, promote Starlink to sole
primary WAN, and add T-Mobile Home Internet Backup (cellular) as pure failover
backup.

## Status

**Planning — nothing purchased/signed up yet.** Research done 2026-10-08, revised
same day after confirming how T-Mobile's gateway actually works; see decision
summary below. Depends on [10_STARLINK_MULTIWAN.md](10_STARLINK_MULTIWAN.md), which
is implemented and currently runs Starlink load-balanced 50/50 against Kinetic.

## Goal

- Cancel Kinetic/Windstream (DSL, `em0`, see
  [research/T3200_bridge_mode.md](../research/T3200_bridge_mode.md)).
- Starlink becomes the sole primary WAN (`opt9`/`em2`), not balanced against anything.
- T-Mobile Home Internet Backup provides automatic failover only, via OPNsense's
  own gateway-group logic — idle until Starlink drops, not sharing load day-to-day.

## Hardware & plan decision (revised 2026-10-08)

**Decision: T-Mobile Home Internet Backup**, using T-Mobile's own included gateway
directly — no third-party cellular router. Scott is already a T-Mobile customer, so
this should land at **$10/mo** (vs. $20/mo standalone), with 100 hrs/mo (130GB)
uncapped 5G.

This **replaces the earlier RUTX50 plan**. Researched and confirmed:

- **The T-Mobile gateway has no Ethernet WAN-in port — it's LAN-out only**, like a
  basic cellular modem. It cannot take Starlink as an input and fail over on its
  own; it isn't designed to sit "in front of" another connection.
- **It also has no bridge/IP-passthrough mode** — unlike the Kinetic T3200 pattern
  this doc originally assumed. It's SIM-locked to T-Mobile's own hardware too, so a
  third-party router (RUTX50 or otherwise) can't use the T-Mobile SIM anyway.
- **Conclusion: all failover logic must live in OPNsense**, not in the cellular
  gateway. This is actually the simpler outcome — it's exactly the Tier1/Tier2
  gateway-group design already planned below, just with the T-Mobile gateway's LAN
  port plugged into OPNsense's `CELLULAR` interface instead of a passthrough-mode
  RUTX50.
- **Accept double-NAT** on the cellular path: T-Mobile gateway NATs, then OPNsense
  NATs again behind it. Fine for an idle, outbound-only failover link (no inbound
  hosting over cellular). OPNsense's `CELLULAR` interface will get a private DHCP
  address from the T-Mobile gateway — apply the same "uncheck block private/bogon
  networks" fix already used for Starlink's CGNAT'd `opt9`.
- This also **eliminates the RUTX50 purchase** (~$300-350 saved); the earlier
  hardware comparison table (RUTX50, Peplink, GL.iNet, ConnecTen, etc.) is now
  moot for this plan and kept below only for reference should T-Mobile Home
  Internet Backup fall through (e.g. poor signal at the house).

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

**Still to confirm:** whether the $10/mo voice-line discount applies automatically
to Scott's existing T-Mobile account/line, or requires a separate sign-up step.
Also confirm signal strength at the house before committing (fallback: Verizon/AT&T
if T-Mobile signal is weak there).

## OPNsense changes required (not yet implemented)

This is a restructuring of the existing Starlink multi-WAN setup
([10_STARLINK_MULTIWAN.md](10_STARLINK_MULTIWAN.md)), not just an addition:

- **Retire `em0` as Kinetic WAN**, repurpose as `CELLULAR` interface once the
  T-Mobile gateway's LAN port is wired in. It will hand back a private DHCP
  address (double-NAT, confirmed above) — apply the same "uncheck block
  private/bogon networks" gotcha documented for Starlink's `opt9`.
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

- [ ] Test T-Mobile cellular signal strength at the house — confirm before signing
      up (fallback to Verizon/AT&T-based option if weak).
- [ ] Sign up for T-Mobile Home Internet Backup; confirm $10/mo (not $20/mo)
      applies to Scott's existing account/line.
- [ ] Cancel Kinetic/Windstream — confirm no contract/ETF penalty first.
- [ ] Wire T-Mobile gateway's LAN port into freed `em0` (no passthrough config —
      it's a standalone double-NAT device; see decision above).
- [ ] Rebuild OPNsense gateway/group config per above.
- [ ] Decide on `gw_switch_default` fix for router's own traffic.
- [ ] Live failover test (pull Starlink, confirm cellular picks up) — same
      cable-pull verification pattern used for Starlink/Kinetic in
      `10_STARLINK_MULTIWAN.md`.
- [ ] Update `STRUCTURE.md`'s WAN description once implemented.
