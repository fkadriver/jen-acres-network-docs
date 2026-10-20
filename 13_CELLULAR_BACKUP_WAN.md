# Cellular Backup WAN (Planned)

Tracking the plan to drop Kinetic/Windstream entirely, promote Starlink to sole
primary WAN, and add T-Mobile Home Internet Backup (cellular) as pure failover
backup.

## Status

**Planning — nothing purchased/signed up yet.** Research done 2026-10-08, revised
same day after confirming how T-Mobile's gateway actually works; see decision
summary below.

**Update 2026-10-20: Kinetic has been turned off**, ahead of this plan's full
implementation. The 50/50 `WAN_BALANCE` group this doc originally described as
"currently running" has been removed — Starlink is now the sole WAN with no gateway
group at all (see [10_STARLINK_MULTIWAN.md](10_STARLINK_MULTIWAN.md)), rather than
the Tier1(Starlink)/Tier2(cellular) failover group this doc plans for below. That
failover group still needs to be built once the T-Mobile gateway is actually wired
in — removing the 50/50 group doesn't get ahead of that step, it just reflects that
balancing against a now-dead Kinetic link stopped making sense before cellular was
ready to take its place.

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

Signal strength at the house confirmed acceptable (2026-10-08).

**Confirmed directly by Scott (2026-10-20):** connection runs through T-Mobile's
gateway (he refers to it as the "4G router"); it only kicks in when the main
connection (Starlink) is down — automatic failover only, never shares load, matching
the Tier1/Tier2 plan below; capped at 100 hours/month, matching the 100 hrs/mo (130GB)
figure above.

**Still to confirm:** whether the $10/mo voice-line discount applies automatically
to Scott's existing T-Mobile account/line, or requires a separate sign-up step.

## OPNsense changes required (not yet implemented)

This is a restructuring of the existing Starlink multi-WAN setup
([10_STARLINK_MULTIWAN.md](10_STARLINK_MULTIWAN.md)), not just an addition:

- **`em0` is freed up (Kinetic turned off 2026-10-20)** — repurpose as `CELLULAR` interface once the
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

- [x] Test T-Mobile cellular signal strength at the house — confirmed acceptable
      (2026-10-08).
- [ ] Sign up for T-Mobile Home Internet Backup; confirm $10/mo (not $20/mo)
      applies to Scott's existing account/line.
- [x] Kinetic/Windstream turned off (2026-10-20).
- [ ] Wire T-Mobile gateway's LAN port into freed `em0` (no passthrough config —
      it's a standalone double-NAT device; see decision above).
- [ ] Rebuild OPNsense gateway/group config per above.
- [ ] Decide on `gw_switch_default` fix for router's own traffic.
- [ ] Live failover test (pull Starlink, confirm cellular picks up) — same
      cable-pull verification pattern used for Starlink/Kinetic in
      `10_STARLINK_MULTIWAN.md`.
- [ ] Update `STRUCTURE.md`'s WAN description once implemented.
