# Starlink Multi-WAN (Load Balanced)

Adding a Starlink V5 (100Mbps) as a second WAN on `em2`, load-balanced 50/50 against
the existing primary WAN (`em0`) since both connections are the same speed.

## Prerequisites

- Starlink router set to **Bypass mode** (IP Passthrough) via the Starlink app before
  wiring anything to OPNsense. Without this you get double-NAT and OPNsense won't see
  a real WAN-facing address.
- Expect a **CGNAT address (100.64.0.0/10)** from Starlink even in bypass mode — this
  is normal and handled in Step 2 below.
- `em2` is free and unused (see [01_OPNSENSE_INSTALLATION.md](01_OPNSENSE_INSTALLATION.md)) —
  no hardware changes needed, just cabling.

## Step 1 — Wire and Assign the Interface

Connect the Starlink dish's Ethernet adapter (or the Starlink router in bypass) to `em2`.

**Navigation**: Interfaces → Assignments

Select `em2` in the "New interface" dropdown → **+ Add** → **Save**.

## Step 2 — Configure the Starlink Interface

**Navigation**: Interfaces → [OPT — the new em2 interface]

| Setting | Value |
|---|---|
| Enable | ✓ |
| Description | `STARLINK` |
| IPv4 Configuration Type | DHCP |
| Block private networks | ☐ off |
| Block bogon networks | ☐ off |

> Unchecking both blocks is required — Starlink's CGNAT range (100.64.0.0/10) reads as
> private/bogon space to OPNsense and the interface will silently lose connectivity
> otherwise. This is the most common Starlink+OPNsense gotcha.

Click **Save** → **Apply Changes**.

## Step 3 — Configure the Gateway

A `STARLINK_DHCP` gateway should auto-create once the interface picks up a DHCP lease.

**Navigation**: System → Gateways → Single

Edit `STARLINK_DHCP`:

| Setting | Value |
|---|---|
| Monitor IP | `1.1.1.1` (don't rely on the DHCP-supplied gateway IP — CGNAT gateways can be unreliable to monitor directly) |
| Weight | `1` (equal to `WAN_DHCP` — both links are 100Mbps, so a straight 50/50 split is correct) |
| Mark as Default Gateway | ☐ off |

Confirm `WAN_DHCP` also has Weight `1`.

## Step 4 — Create the Gateway Group

**Navigation**: System → Gateways → Group → **+**

| Setting | Value |
|---|---|
| Group Name | `WAN_BALANCE` |
| Gateway: WAN_DHCP | Tier 1 |
| Gateway: STARLINK_DHCP | Tier 1 |

Equal tiers on both members is what makes this a load-balance (shared) group instead
of a failover group — different tiers would only fail over, never share.

## Step 5 — Outbound NAT

**Navigation**: Firewall → NAT → Outbound

Switch mode from Automatic to **Hybrid**, and confirm outbound NAT rules exist
translating LAN/VLAN subnets out `STARLINK` as well as `WAN` — automatic mode only
reliably covers the original WAN interface.

## Step 6 — Point Traffic at the Gateway Group

Creating the group does nothing on its own — each firewall rule's **Gateway** field
must be changed from the default to `WAN_BALANCE` for policy routing to apply.

**Navigation**: Firewall → Rules → [LAN / SERVERS / WiFI_Secure / GUEST / etc.]

For the general internet-access Pass rules on each interface, set Gateway = `WAN_BALANCE`.

**Exceptions — keep these pinned to a single gateway:**

- Tailscale / management traffic — mid-session gateway changes can break `tailscaled`'s
  established connections. Keep the flat-tailnet posture's Tailscale-only MGMT rules
  pointed at a single fixed gateway (see [05_FIREWALL_RULES.md](05_FIREWALL_RULES.md)).
- Any real-time traffic (VoIP, gaming) that's latency-sensitive — Starlink's satellite
  path has materially higher latency/jitter than a terrestrial link, so pin these to
  the primary WAN with a specific rule above the balanced default.

## Step 7 — Sticky Connections

**Navigation**: System → Settings → Advanced → Firewall/NAT tab

Enable **Sticky Connections**. This pins an already-established session to whichever
gateway it started on, instead of letting individual new states within the same
connection split across both links — important for anything stateful.

Click **Save**.

## Verification

- [ ] `STARLINK` interface shows a DHCP lease and "up" status (Interfaces → Overview)
- [ ] Both `WAN_DHCP` and `STARLINK_DHCP` show green/online (System → Gateways → Single)
- [ ] `WAN_BALANCE` group shows both members active (System → Gateways → Group)
- [ ] From a LAN client, repeated `curl https://ifconfig.me` (or similar) over many
      requests shows both the primary WAN IP and a Starlink-range IP appearing
- [ ] Pull the Starlink cable — LAN clients on the balanced rule stay online via
      primary WAN only (confirms failover-within-balance works)
- [ ] Tailscale/SSH sessions survive a Starlink flap without dropping (confirms the
      pinned-gateway exception in Step 6 is working)
