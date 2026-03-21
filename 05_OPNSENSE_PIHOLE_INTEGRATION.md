# OPNsense + Pi-hole Integration Guide

## Overview

This guide configures Pi-hole on a Raspberry Pi 4 in the SERVERS network (192.168.10.10) to provide DNS filtering with full source IP visibility for tracking which devices make DNS requests.

## Architecture

```
┌───────────────────────────────────────────────────────────────┐
│  OPNsense Router (Protectli FW41)                             │
│  - Dnsmasq DHCP server for all networks                       │
│  - Firewall rules allow DNS to Pi-hole                        │
│  - DNS Resolver disabled (won't interfere with Pi-hole)       │
└───────────────┬───────────────────────────────────────────────┘
                │
    ┌───────────┼───────────┬─────────────┐
    │           │           │             │
   em1         em1         em2           em3
 (VLAN 1)   (VLAN 10)   (dedicated)   (dedicated)
  MGMT       SERVERS    WIFI_SECURE     GUEST
192.168.1.x 192.168.10.x 192.168.11.x 192.168.20.x
                │
        ┌───────┴───────┐
        │               │
┌───────▼───────┐ ┌─────▼─────────┐
│   Pi-hole     │ │  Pi-hole      │
│   Primary     │ │  Backup       │
│   RPi4        │ │  RPi3         │
│ 192.168.10.10 │ │ 192.168.10.11 │
│               │ │               │
│ DNS Port 53   │ │ DNS Port 53   │
│ Web Port 80   │ │ Web Port 80   │
└───────┬───────┘ └───────┬───────┘
        │                 │
        └────────┬────────┘
                 ▼
         Upstream DNS:
         - 1.1.1.1 (Cloudflare)
         - 8.8.8.8 (Google)
```

## Benefits of This Setup

✅ **Full Source IP Visibility**: Pi-hole sees the actual device IPs, not just the router IP
✅ **Per-Device Tracking**: Can identify which device made which DNS request
✅ **VLAN Identification**: Source IP reveals which VLAN the request came from
✅ **Centralized Management**: Dual Pi-holes serve all VLANs with redundancy
✅ **High Availability**: Backup Pi-hole ensures DNS continues if primary fails
✅ **Selective Filtering**: Can apply different blocking lists per VLAN (via Pi-hole groups)

## Prerequisites

- [ ] OPNsense installed with networks configured (see [02_VLAN_CONFIG.md](02_VLAN_CONFIG.md))
- [ ] Raspberry Pi 4 with Raspberry Pi OS Lite (primary Pi-hole)
- [ ] Raspberry Pi 3 with Raspberry Pi OS Lite (backup Pi-hole, optional)
- [ ] Pi-hole installed on both Pis (see [04_PIHOLE_SETUP.md](04_PIHOLE_SETUP.md))
- [ ] RPi4 connected to switch port 2 on VLAN 10 (Servers)
- [ ] RPi3 connected to switch port 3 on VLAN 10 (Servers) - if using backup

---

## Part 1: Pi-hole Setup on Raspberry Pi 4

For complete Pi-hole installation including pre-boot SD card configuration and Tailscale setup, see: **[04_PIHOLE_SETUP.md](04_PIHOLE_SETUP.md)**

Quick summary:

**Primary Pi-hole (RPi4):**
- **IP Address**: 192.168.10.10 (static)
- **Gateway**: 192.168.10.1
- **DNS during setup**: 1.1.1.1, 8.8.8.8
- **Web Interface**: http://192.168.10.10/admin

**Backup Pi-hole (RPi3) - Optional:**
- **IP Address**: 192.168.10.11 (static)
- **Gateway**: 192.168.10.1
- **DNS during setup**: 1.1.1.1, 8.8.8.8
- **Web Interface**: http://192.168.10.11/admin

See [04_PIHOLE_SETUP.md](04_PIHOLE_SETUP.md) for complete setup including Gravity Sync for configuration synchronization.

---

## Part 2: OPNsense Configuration

### Step 1: Disable OPNsense DNS Services (Use Pi-hole Instead)

**Why**: We want devices to use Pi-hole directly, not OPNsense's DNS services. Dnsmasq will handle DHCP only.

**Disable Unbound DNS:**

**Navigation**: Services → Unbound DNS → General

- **Enable**: ✗ Uncheck "Enable Unbound"

Click **Save** → **Apply**

**Configure Dnsmasq for DHCP Only (no DNS):**

**Navigation**: Services → Dnsmasq DNS & DHCP → General

**IMPORTANT**: Change **Advanced options** (top left) to **"Advanced"** to reveal all settings.

- **Enable**: ✓ Checked (we need it for DHCP)
- **Listen port**: `0` (disables DNS functionality)
- **Interface [no DHCP]**: Nothing selected (ensures DHCP works on all interfaces with ranges)

Click **Save** → **Apply**

### Step 2: Configure DHCP to Use Pi-hole

Dnsmasq DHCP uses **DHCP options** to set DNS servers (not in the range config).

**Navigation**: Services → Dnsmasq DNS & DHCP → DHCP options

Click **+** to add a DNS server option:

| Setting | Value |
|---------|-------|
| **Interface** | Any |
| **Type** | Set |
| **Option6** | dns-server [23] |
| **Value** | 192.168.10.10,192.168.10.11 |
| **Description** | Pi-hole DNS servers |

Click **Save** → **Apply**

**Note**: Option6 `dns-server [23]` sets DNS servers for all DHCP clients. If you're not using a backup Pi-hole, only enter 192.168.10.10.

#### MGMT
DHCP not enabled (static IPs), but devices should manually configure:
- Primary DNS: 192.168.10.10
- Secondary DNS: 192.168.10.11 (backup Pi-hole)

### Step 3: Configure Firewall Rules for DNS

**Navigation**: Firewall → Rules → [Interface]

For each VLAN, ensure DNS traffic to Pi-hole is allowed:

#### Rule: Allow DNS to Pi-hole

**For SERVERS, WIFI_SECURE, GUEST:**

- **Action**: Pass
- **Interface**: [Current Interface]
- **Protocol**: TCP/UDP
- **Source**: [Interface] net
- **Destination**: Single host - 192.168.10.10
- **Destination port range**: DNS (53)
- **Description**: Allow DNS queries to Pi-hole

Click **Save** → **Apply Changes**

**IMPORTANT**: This rule must be placed BEFORE any block rules for isolated VLANs.

### Step 4: Verify OPNsense Configuration

**Check DNS Server in DHCP Leases:**

Navigation: Services → Dnsmasq DNS & DHCP → Leases

- Verify devices are receiving DNS: 192.168.10.10

**Test from OPNsense itself:**

Navigation: Interfaces → Diagnostics → DNS Lookup

- Hostname: google.com
- Server: 192.168.10.10
- Click **Lookup**

Should resolve successfully.

---

## Part 3: Pi-hole Configuration for Multi-VLAN

### Step 1: Configure Upstream DNS

**Navigation**: Settings → DNS

**Upstream DNS Servers:**
- ✓ Cloudflare: 1.1.1.1
- ✓ Cloudflare (IPv6): 2606:4700:4700::1111
- ✓ Google: 8.8.8.8
- ✓ Google (IPv6): 2001:4860:4860::8888

**Interface settings:**
- **Interface listening behavior**: Listen on all interfaces
  - This allows Pi-hole to accept queries from all VLANs

Click **Save**

### Step 2: Configure Conditional Forwarding (Optional)

This enables Pi-hole to resolve local hostnames.

**Navigation**: Settings → DNS → Advanced DNS Settings

**Conditional forwarding:**
- ✓ Use Conditional Forwarding
- **Local network in CIDR notation**: 192.168.0.0/16
  - This covers all your network ranges (1.x, 10.x, 11.x, 20.x)
- **IP address of DHCP server (router)**: 192.168.10.1 (OPNsense SERVERS interface)
- **Local domain name**: lan

Click **Save**

### Step 3: Enable Query Logging

**Navigation**: Settings → Privacy

- **Privacy level**: Show everything
  - Required to see source IPs and domains
- **Max log age**: 365 days (adjust as needed)

Click **Save**

### Step 4: Configure Blocklists

**Navigation**: Group Management → Adlists

Default lists are already configured. To add more:

**Recommended Additional Lists:**
- **Hagezi Pro**: `https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/domains/pro.txt`
- **OISD Full**: `https://big.oisd.nl/`
- **StevenBlack**: `https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts`

Click **Add** for each list, then:

**Navigation**: Tools → Update Gravity

Click **Update** to download and compile blocklists.

---

## Part 4: Testing DNS Resolution

### From OPNsense

**Navigation**: Interfaces → Diagnostics → DNS Lookup

- Hostname: google.com
- Server: 192.168.10.10
- Result: Should resolve

### From Each VLAN

Connect a device to each VLAN and test:

```bash
# Check DNS server
nslookup google.com

# Should show server: 192.168.10.10

# Test blocked domain
nslookup doubleclick.net

# Should show 0.0.0.0 or be blocked
```

### From Pi-hole Web Interface

**Navigation**: Dashboard

You should see:
- **Queries**: Increasing as devices make DNS requests
- **Clients**: Showing devices from all VLANs
- **Client IPs**: Showing actual device IPs (192.168.X.Y), not router IP

**Navigation**: Query Log

Click on any query to see:
- **Client**: Source IP address (e.g., 192.168.200.105)
- **Domain**: Domain being requested
- **Type**: A, AAAA, PTR, etc.
- **Status**: Allowed, Blocked, etc.
- **Time**: Timestamp

---

## Part 5: Advanced Pi-hole Configuration

### Create Groups for VLAN-Specific Filtering

You can apply different blocklists to different VLANs.

**Example: Block social media on GUEST network only**

#### Step 1: Create Client Group

**Navigation**: Group Management → Groups

- **Name**: Guest Network
- **Description**: Devices on GUEST network
- Click **Add**

#### Step 2: Assign Clients to Group

**Navigation**: Group Management → Clients

For each device on GUEST network (192.168.20.x):
- **IP address**: 192.168.20.0/24 (entire subnet)
- **Group assignment**: Guest Network
- Click **Add**

#### Step 3: Create/Assign Blocklist

**Navigation**: Group Management → Adlists

Add social media blocklist:
- **Address**: `https://raw.githubusercontent.com/StevenBlack/hosts/master/alternates/fakenews-gambling-porn-social/hosts`
- **Group assignment**: Guest Network
- Click **Add**

**Update Gravity**: Tools → Update Gravity → Update

Now social media will be blocked only for GUEST network devices.

### Create Custom Blocklists

**Navigation**: Group Management → Domains

**Add domains to block:**
- Domain: example-ads.com
- Type: Exact blocking
- Group: Default (all) or specific group
- Click **Add**

### Whitelist Domains

**Navigation**: Whitelist → Exact

Add domains that should never be blocked:
- Domain: important-service.com
- Click **Add**

---

## Part 6: Monitoring and Maintenance

### View Query Statistics

**Navigation**: Dashboard

Real-time stats:
- Queries in last 24 hours
- Queries blocked
- Percent blocked
- Unique domains
- Unique clients

### Top Clients

**Navigation**: Dashboard → Top Clients

See which devices (by IP) are making the most DNS requests.

**Identify Networks by IP:**
- 192.168.1.x = MGMT
- 192.168.10.x = SERVERS
- 192.168.11.x = WIFI_SECURE
- 192.168.20.x = GUEST

### Top Domains

**Navigation**: Dashboard → Top Permitted Domains / Top Blocked Domains

See most frequently accessed/blocked domains.

### Long-term Statistics

**Navigation**: Long-term Data → Query Log

Export data for analysis:
- **Database location**: `/etc/pihole/pihole-FTL.db`
- Use SQLite tools to query custom reports

### Update Pi-hole

```bash
pihole -up
```

Regular updates ensure:
- Latest security patches
- New blocklist formats
- Bug fixes

### Backup Pi-hole Configuration

**Navigation**: Settings → Teleporter

**Export:**
- Click **Backup** to download configuration
- Saves: settings, whitelist, blacklist, adlists, groups

**Import:**
- Click **Choose File** to upload previous backup
- Click **Restore** to apply

---

## Part 7: Troubleshooting

### Devices not using Pi-hole

**Check:**
1. DHCP settings in OPNsense provide correct DNS (192.168.10.10)
2. Device has received new DHCP lease (renew or reconnect)
3. Device not using hardcoded DNS (check network settings)

**Test from device:**
```bash
nslookup google.com
# Server should show 192.168.10.10
```

### Pi-hole not blocking ads

**Check:**
1. Gravity updated: Tools → Update Gravity
2. Query logged: Settings → Privacy → Show everything
3. Blocklists active: Group Management → Adlists (green checkmarks)
4. Domain not whitelisted

**Test:**
```bash
nslookup doubleclick.net
# Should show 0.0.0.0 or be blocked
```

### Cannot access Pi-hole from certain VLANs

**Check OPNsense firewall rules:**
- Navigation: Firewall → Rules → [VLAN]
- Ensure rule allows TCP/UDP port 53 to 192.168.10.10
- Ensure rule allows TCP port 80 for web interface (if needed)

### High DNS latency

**Check:**
1. Pi-hole upstream DNS servers responding quickly
   - Settings → DNS → Test each upstream server
2. Pi-hole not overloaded
   - Dashboard → System stats (CPU, memory)
3. Network connectivity to Pi-hole stable
   - Ping test from OPNsense: Interfaces → Diagnostics → Ping

### Pi-hole web interface not accessible

**From Pi-hole itself:**
```bash
sudo service lighttpd status
# Should show "active (running)"

# If not running:
sudo service lighttpd start
```

**Check firewall on Pi-hole:**
```bash
sudo iptables -L
# Should allow port 80
```

---

## Part 8: Optional Enhancements

### 1. Enable HTTPS for Pi-hole Web Interface

```bash
# Install certbot
sudo apt install certbot -y

# Generate self-signed cert (or use Let's Encrypt if you have a domain)
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/lighttpd/server.key \
  -out /etc/lighttpd/server.crt

# Configure lighttpd for SSL
sudo nano /etc/lighttpd/external.conf
```

Add:
```
$SERVER["socket"] == ":443" {
    ssl.engine = "enable"
    ssl.pemfile = "/etc/lighttpd/combined.pem"
}
```

Combine cert and key:
```bash
sudo cat /etc/lighttpd/server.crt /etc/lighttpd/server.key | \
  sudo tee /etc/lighttpd/combined.pem
sudo chmod 600 /etc/lighttpd/combined.pem
sudo service lighttpd restart
```

Access: `https://192.168.10.10/admin`

### 2. Configure DHCP Static Lease for Pi-hole

**In OPNsense:**

Navigation: Services → Dnsmasq DNS & DHCP → Hosts

- **MAC address**: (Pi's MAC address - check with `ip addr` on Pi)
- **IP address**: 192.168.10.10
- **Hostname**: pihole
- **Description**: Pi-hole DNS Server
- Click **Save** → **Apply**

This ensures Pi-hole always gets the same IP even if DHCP renews.

### 3. Add Redundant Pi-hole (Optional)

For high availability, run a second Pi-hole on 192.168.10.11:

**In OPNsense Dnsmasq DHCP:**
- **DNS servers**: 192.168.10.10, 192.168.10.11

Both Pi-holes will serve DNS, providing redundancy if one fails.

### 4. Monitor Pi-hole with Grafana

Install Grafana and InfluxDB to create beautiful dashboards:
- Query statistics over time
- Client activity graphs
- Block rate trends

See: https://github.com/eko/pihole-exporter

---

## Configuration Checklist

- [ ] Pi-hole installed on RPi4 at 192.168.10.10 (see [04_PIHOLE_SETUP.md](04_PIHOLE_SETUP.md))
- [ ] Static IP configured on Pi
- [ ] OPNsense Dnsmasq DHCP provides Pi-hole as DNS for all networks
- [ ] OPNsense DNS Resolver/Forwarder disabled
- [ ] Firewall rules allow DNS to Pi-hole from all networks
- [ ] Pi-hole listening on all interfaces
- [ ] Blocklists updated (Gravity)
- [ ] Query logging enabled
- [ ] Conditional forwarding configured for hostname resolution
- [ ] Tested DNS resolution from each network
- [ ] Verified source IPs visible in Pi-hole logs
- [ ] Backed up Pi-hole configuration (Teleporter)
- [ ] (Optional) Tailscale configured for remote access

---

## Summary

You now have:
✅ Pi-hole serving DNS for all networks
✅ Full visibility into which devices make DNS requests
✅ Per-device and per-network tracking
✅ Centralized ad/tracker blocking
✅ Firewall rules protecting Pi-hole access
✅ Flexible configuration for network-specific filtering
✅ Hostname resolution via conditional forwarding

The key advantage of this setup: **Pi-hole sees the actual client IPs** (192.168.X.Y) rather than just the router IP, giving you complete visibility into DNS activity across your entire network.
