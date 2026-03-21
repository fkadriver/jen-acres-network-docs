# Pi-hole Setup

## Overview

Two Pi-holes provide DNS filtering and ad-blocking for all VLANs. Pi-hole is installed and
configured via **NixOS** — see the `nixos/` directory for declarative configuration. This
document covers what NixOS manages, the OPNsense-side integration steps that require manual
configuration, and verification.

**Complete after**: [05_FIREWALL_RULES.md](05_FIREWALL_RULES.md)

---

## Hardware

| Role | Device | OS | IP | Switch Port | Tailscale Name |
|------|--------|----|----|-------------|----------------|
| **Primary** | Raspberry Pi 4 | NixOS | 192.168.10.10 | SW01 Port 2 → VLAN 10 | `pihole-primary` |
| **Backup** | Raspberry Pi 3 | NixOS | 192.168.10.11 | SW01 Port 3 → VLAN 10 | `pihole-backup` |

Both units land on VLAN 10 (SERVERS, 192.168.10.0/24). Gateway: 192.168.10.1

---

## Architecture

```
  OPNsense (Dnsmasq DHCP — port 0 disables DNS)
       │  DHCP option 6: 192.168.10.10, 192.168.10.11
       │
  ┌────┴────────────────┐
  │                     │
Pi-hole Primary     Pi-hole Backup
192.168.10.10       192.168.10.11
  │                     │
  └────────┬────────────┘
           │ Upstream DNS
       1.1.1.1 / 8.8.8.8
```

Pi-hole sees **actual client IPs** (not the router IP) because Dnsmasq DHCP hands out the Pi-hole
addresses directly — clients query Pi-hole directly, bypassing OPNsense.

---

## What NixOS Manages

The following are declared in `nixos/` and applied with `nixos-rebuild switch`:

- Pi-hole installation and service management
- Static IP addresses (192.168.10.10 / .11)
- DNS listening mode (`ALL` — required for cross-VLAN access)
- Upstream DNS servers (1.1.1.1, 8.8.8.8)
- Conditional forwarding: `192.168.0.0/16 → 192.168.10.1`
- Tailscale installation and configuration (`--accept-routes --ssh`)
- Gravity Sync (blocklist sync between primary and backup)
- Blocklist sources

> Do not configure these settings via the Pi-hole web UI — NixOS will overwrite them on
> the next rebuild.

---

## Part 1: OPNsense Integration

### Step 1: Disable Unbound DNS

Pi-hole handles all DNS. Unbound must be disabled so it does not compete on port 53.

**Navigation**: Services → Unbound DNS → General

- **Enable**: ✗ (uncheck)

Click **Save** → **Apply**

> **Does disabling Unbound affect Dnsmasq DHCP?**
>
> No. They are completely independent:
>
> | Service | Ports | Affected by Unbound state? |
> |---------|-------|---------------------------|
> | Dnsmasq DHCP | UDP 67/68 | Never |
> | Dnsmasq DNS | UDP/TCP 53 (disabled — port=0) | No conflict |
> | Unbound DNS | UDP/TCP 53 | — |
>
> DHCP leases, static mappings, and DHCP options are unaffected regardless of Unbound's state.

### Step 2: Configure Dnsmasq for DHCP Only

**Navigation**: Services → Dnsmasq DNS & DHCP → General

Enable **Advanced** mode (top-left toggle).

| Setting | Value |
|---------|-------|
| Enable | ✓ |
| Listen port | `0` ← disables DNS, keeps DHCP |

Click **Save** → **Apply**

> **⚠️ 26.1 upgrade note**: OPNsense 26.1 changes the default IPv6 mode to use Dnsmasq.
> After upgrading, verify this listen port is still `0`. If it was reset to `53`, Pi-hole
> DNS is bypassed. Check immediately after any firmware upgrade.

### Step 3: Set DHCP DNS Option

Tell DHCP clients to use both Pi-holes for DNS.

**Navigation**: Services → Dnsmasq DNS & DHCP → DHCP options → **+**

| Setting | Value |
|---------|-------|
| Interface | Any |
| Type | Set |
| Option | `dns-server [6]` |
| Value | `192.168.10.10,192.168.10.11` |
| Description | Pi-hole DNS servers (primary + backup) |

Click **Save** → **Apply**

### Step 4: Create Pi_hole_DNS Alias

**Navigation**: Firewall → Aliases → **+**

| Setting | Value |
|---------|-------|
| Name | `Pi_hole_DNS` |
| Type | Host(s) |
| Content | `192.168.10.10`, `192.168.10.11` |
| Description | Pi-hole DNS servers |

Click **Save** → **Apply Changes**

This alias is referenced in the firewall rules (see [05_FIREWALL_RULES.md](05_FIREWALL_RULES.md)).

### Step 5: Verify DHCP Static Leases

Pi-holes use static IPs set by NixOS, but adding static leases in OPNsense improves hostname
visibility in DHCP logs.

**Navigation**: Services → Dnsmasq DNS & DHCP → Hosts → **+**

| Field | Primary | Backup |
|-------|---------|--------|
| MAC | (from `ip link show eth0` on Pi) | — |
| IP | `192.168.10.10` | `192.168.10.11` |
| Hostname | `pihole-primary` | `pihole-backup` |

Click **Save** → **Apply**

### Step 6: Enable Host Discovery (26.1)

OPNsense 26.1 adds automatic host discovery to help resolve DHCP hostnames in Pi-hole logs.

**Navigation**: Services → Host Discovery

- **Enable**: ✓
- **Scan network**: `192.168.0.0/20` (Secure_Net supernet)

Click **Save** → **Apply**

This supplements Pi-hole's conditional forwarding — hostname data pushed to OPNsense's
internal DNS becomes resolvable via the `192.168.0.0/16 → 192.168.10.1` conditional
forwarding rule already configured in NixOS.

---

## Part 2: Tailscale Admin Configuration

After Pi-holes join the tailnet (handled by NixOS):

1. Go to `https://login.tailscale.com/admin/machines`
2. For each Pi-hole:
   - Verify machine name matches `pihole-primary` / `pihole-backup`
   - **Edit tags** → add `tag:server`
   - **Disable key expiry**

### Configure Pi-hole as Tailscale DNS

**Navigation**: Tailscale Admin → DNS → Settings

- **Global nameserver**: `192.168.10.10` (Pi-hole primary, accessible via subnet route)
- **Override local DNS**: ✓

This routes all Tailscale device DNS queries through Pi-hole even when remote.

---

## Part 3: Pi-hole Web Configuration

> Most settings are managed by NixOS. These are the items that need verification or
> occasional manual adjustment via the web UI.

Access via Tailscale:

| Pi-hole | URL |
|---------|-----|
| Primary | `http://pihole-primary.warthog-royal.ts.net/admin` |
| Backup | `http://pihole-backup.warthog-royal.ts.net/admin` |

Or from SERVERS VLAN: `http://192.168.10.10/admin`

### Verify DNS Listening Mode

**Navigation**: Settings → DNS → Interface settings

- Should show **"Permit all origins"** (set by NixOS as `listeningMode = "ALL"`)

### Verify Conditional Forwarding

**Navigation**: Settings → DNS → Advanced DNS settings → Reverse server

Should show an entry for `192.168.0.0/16` pointing to `192.168.10.1`.

This allows Pi-hole to resolve PTR queries (reverse DNS) through OPNsense, which gives
client hostnames in query logs instead of bare IPs.

### Add/Update Blocklists

**Navigation**: Group Management → Adlists

Recommended lists (if not already in NixOS config):

| Name | URL |
|------|-----|
| Hagezi Pro | `https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/domains/pro.txt` |
| OISD Full | `https://big.oisd.nl/` |
| StevenBlack | `https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts` |

After adding: **Navigation**: Tools → Update Gravity → **Update**

### Pi-hole Groups for VLAN-Specific Filtering

Pi-hole groups allow different blocking rules per VLAN.

**Example: Block social media on GUEST only**

1. **Group Management → Groups**: Create group `Guest Network`
2. **Group Management → Clients**: Add `192.168.20.0/24` → assign to `Guest Network`
3. **Group Management → Adlists**: Add social media blocklist → assign to `Guest Network`
4. **Tools → Update Gravity**

SERVERS/WIFI_SECURE clients remain in `Default` group with standard blocking.

---

## Verification

### Test from OPNsense

**Navigation**: Interfaces → Diagnostics → DNS Lookup

- Hostname: `google.com`
- Server: `192.168.10.10`
- Should resolve successfully

Or via SSH:
```bash
ssh scott@opnsense.warthog-royal.ts.net
drill google.com @192.168.10.10    # primary
drill google.com @192.168.10.11    # backup
```

### Test from a Client on Each VLAN

```bash
# Check DNS server assigned by DHCP
nslookup google.com
# Server should show 192.168.10.10

# Test ad blocking
nslookup doubleclick.net
# Should return 0.0.0.0 or NXDOMAIN
```

### Verify Source IPs in Pi-hole

**Navigation**: Dashboard → Query Log

Client IPs should show as `192.168.X.Y` (actual device IPs), not `192.168.1.1` (router IP).
If you see only the router IP, Dnsmasq DNS listen port is not `0` — check Step 2 above.

### Test DNS Failover

```bash
# Disable primary temporarily
ssh pihole-primary sudo pihole disable

# From a client — should still resolve via backup (192.168.10.11)
nslookup google.com

# Re-enable primary
ssh pihole-primary sudo pihole enable
```

### Verify Gravity Sync (Backup)

```bash
ssh pihole-backup gravity-sync status
gravity-sync logs
```

---

## Checklist

```
[ ] Unbound DNS disabled on OPNsense
[ ] Dnsmasq listen port = 0 (DHCP only)
[ ] DHCP option 6 = 192.168.10.10, 192.168.10.11
[ ] Pi_hole_DNS alias created
[ ] Host Discovery enabled (26.1)
[ ] Pi-hole primary resolves DNS (drill google.com @192.168.10.10)
[ ] Pi-hole backup resolves DNS (drill google.com @192.168.10.11)
[ ] Source IPs visible in Pi-hole query log (device IPs, not router IP)
[ ] Tailscale admin: pihole-primary tagged tag:server, key expiry disabled
[ ] Tailscale admin: pihole-backup tagged tag:server, key expiry disabled
[ ] Tailscale DNS configured: global nameserver 192.168.10.10
[ ] Gravity Sync running (blocklists synced primary → backup)
```

---

## Troubleshooting

**Devices not using Pi-hole**
1. Renew DHCP lease on device (`dhclient` or disconnect/reconnect)
2. Check `nslookup google.com` — server should be 192.168.10.10
3. Verify DHCP option 6 is set in Dnsmasq (Step 3)
4. Check firewall allows UDP/TCP 53 from client VLAN to 192.168.10.10

**Pi-hole only shows router IP in query log (not device IPs)**
1. Dnsmasq listen port must be `0` — if it's `53`, OPNsense is intercepting DNS queries
2. Navigation: Services → Dnsmasq → General → Advanced → Listen port = `0`

**Dnsmasq listen port reset after 26.1 upgrade**
- Check immediately after upgrade: Services → Dnsmasq → General → Advanced → Listen port
- If reset to `53`, set back to `0` and Save → Apply

**Pi-hole not blocking**
1. Run gravity update: `sudo pihole -g` (or Tools → Update Gravity in web UI)
2. Verify blocklists show green checks: Group Management → Adlists
3. Check domain is not whitelisted

**Cannot reach Pi-hole web UI**
```bash
# Check lighttpd service
ssh pihole-primary sudo systemctl status lighttpd
sudo systemctl start lighttpd
```

**Conditional forwarding not resolving hostnames**
1. Verify entry in Settings → DNS → Advanced → Reverse server: `192.168.0.0/16 → 192.168.10.1`
2. OPNsense DHCP must have static leases with hostnames set (Services → Dnsmasq → Hosts)
3. 26.1: Enable Host Discovery for better hostname population

---

## Related Docs

- [03_VLAN_CONFIG.md](03_VLAN_CONFIG.md) — DHCP ranges per VLAN
- [05_FIREWALL_RULES.md](05_FIREWALL_RULES.md) — DNS firewall rules (Pi_hole_DNS alias)
- `nixos/` — NixOS configurations for pihole-primary and pihole-backup
