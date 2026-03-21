# OPNsense Tailscale Subnet Router Configuration

## Overview

Configure OPNsense as a Tailscale subnet router to advertise the supernet `192.168.0.0/16`, providing secure remote access to all VLANs from any device on your Tailscale network.

## Architecture

```
Tailscale Network (100.x.x.x)
         │
         │ VPN Connection
         │
    ┌────▼─────────────────┐
    │  OPNsense Router     │
    │  Tailscale Subnet    │
    │  Router              │
    │                      │
    │  Advertises:         │
    │  192.168.0.0/16      │
    └────┬─────────────────┘
         │
    ┌────┴────┬────────┬─────────┐
    │         │        │         │
   MGMT    SERVERS  WIFI_SEC   GUEST
   em1      em1       em2       em3
 (VLAN1) (VLAN10) (dedicated)(dedicated)
```

**Benefits:**
- ✅ Access all VLANs remotely via single subnet route
- ✅ Secure encrypted VPN access (WireGuard-based)
- ✅ No port forwarding required
- ✅ Works behind NAT/CGNAT
- ✅ Cross-platform clients (Windows, Mac, Linux, iOS, Android)

## Prerequisites

- [ ] OPNsense installed and accessible
- [ ] VLANs configured (see [02_VLAN_CONFIG.md](02_VLAN_CONFIG.md))
- [ ] Tailscale account created at https://login.tailscale.com
- [ ] SSH access to OPNsense (or console access)

---

## Part 1: Install Tailscale on OPNsense

### Method 1: Using FreeBSD Packages (Recommended)

OPNsense is based on FreeBSD 14.1, and Tailscale is available in the FreeBSD package repository.

**Step 1: Enable SSH access (if not already enabled)**

Via Web UI:
- **Navigation**: System → Settings → Administration
- **Secure Shell**:
  - ✓ Enable Secure Shell
  - **Listen Interfaces**: LAN
  - **Permit root user login**: Yes (temporarily)
  - **Permit password login**: Yes (temporarily)
- Click **Save**

**Step 2: SSH into OPNsense**

```bash
ssh root@192.168.1.1
```

**Step 3: Install Tailscale**

```bash
# Update package repository
pkg update

# Install Tailscale
pkg install -y tailscale

# Enable Tailscale service
sysrc tailscaled_enable="YES"

# Start Tailscale daemon
service tailscaled start
```

**Verify installation:**
```bash
tailscale version
# Should show: 1.x.x or later
```

---

## Part 2: Authenticate and Configure Tailscale

### Start Tailscale and Authenticate

```bash
# Start Tailscale and authenticate
# Use --advertise-routes to advertise your supernet
tailscale up --advertise-routes=192.168.0.0/16 --accept-routes
```

**Output:**
```
To authenticate, visit:

  https://login.tailscale.com/a/xxxxxxxxxxxxx
```

**From your workstation:**
1. Copy the URL shown
2. Open in browser
3. Login to Tailscale
4. Click **Connect**

**Back on OPNsense SSH session:**
```bash
# Verify connection
tailscale status

# Should show:
# 100.x.x.x   opnsense   youremail@   linux   -
```

### Approve Subnet Routes in Tailscale Admin Console

**Important:** Tailscale requires manual approval for subnet routes.

1. Go to https://login.tailscale.com/admin/machines
2. Find your OPNsense device
3. Click **⋮** (three dots) → **Edit route settings**
4. Under **Subnet routes**, check **192.168.0.0/16**
5. Click **Save**

**Verify route approval:**
```bash
tailscale status

# Should now show:
# 100.x.x.x   opnsense   youremail@   linux   active; relay "xxx"; subnet router
```

---

## Part 3: Configure OPNsense Firewall for Tailscale

### Create Tailscale Interface Alias

**Navigation**: Firewall → Aliases → All

**Create new alias:**
- **Name**: Tailscale_Net
- **Type**: Network(s)
- **Content**: 100.64.0.0/10
  - This is the CGNAT range Tailscale uses for VPN IPs
- **Description**: Tailscale VPN network
- Click **Save** → **Apply**

### Create Firewall Rules to Allow Tailscale Traffic

#### Rule 1: Allow Tailscale on WAN (for initial connection)

**Navigation**: Firewall → Rules → WAN

**Add rule:**
- **Action**: Pass
- **Interface**: WAN
- **Protocol**: UDP
- **Source**: any
- **Destination**: WAN address
- **Destination port**: 41641 (Tailscale default UDP port)
- **Description**: Allow Tailscale UDP
- Click **Save** → **Apply Changes**

#### Rule 2: Allow Tailscale Access to All VLANs

**Navigation**: Firewall → Rules → Floating

**Add floating rule:**
- **Action**: Pass
- **Quick**: ✓ (Apply rule immediately)
- **Interface**: Select all interfaces (MGMT, SERVERS, WIFI_SECURE, GUEST)
- **Direction**: in
- **Protocol**: any
- **Source**: Tailscale_Net (the alias you created)
- **Destination**: any
- **Description**: Allow Tailscale VPN access to all networks
- Click **Save** → **Apply Changes**

**Why floating rule?** This applies the same rule to all interfaces without duplication.

#### Rule 3: Allow Tailscale Interface Traffic

**Navigation**: Interfaces → Assignments

Check if `tailscale0` interface exists. If not listed, it will be created automatically by Tailscale.

**If tailscale0 is assigned as an interface:**

**Navigation**: Firewall → Rules → TAILSCALE0

**Add rule:**
- **Action**: Pass
- **Protocol**: any
- **Source**: any
- **Destination**: any
- **Description**: Allow all traffic on Tailscale interface
- Click **Save** → **Apply Changes**

---

## Part 4: Enable IP Forwarding and Configure Routing

### Enable IP Forwarding

**Navigation**: System → Settings → Tunables

**Check if enabled:**
- Look for `net.inet.ip.forwarding`
- Should be set to `1`

**If not present or set to 0:**
- **Tunable**: net.inet.ip.forwarding
- **Value**: 1
- **Description**: Enable IPv4 forwarding for Tailscale
- Click **Save**

### Verify Kernel IP Forwarding

```bash
# SSH into OPNsense
sysctl net.inet.ip.forwarding

# Should show:
# net.inet.ip.forwarding: 1
```

### Add Static Routes (Optional)

Normally not needed, but if you have routing issues:

**Navigation**: System → Routes → Configuration

**Add route:**
- **Network Address**: 100.64.0.0/10
- **Gateway**: Interface address (select Tailscale0 if available)
- **Description**: Tailscale VPN network
- Click **Save** → **Apply**

---

## Part 5: Configure Tailscale Settings

### Access Tailscale Admin Console

Go to: https://login.tailscale.com/admin/machines

### Recommended Settings for OPNsense Device

**1. Disable Key Expiry**
- Click on your OPNsense device
- **Key expiry**: Click **Disable key expiry**
- Prevents authentication from expiring after 180 days

**2. Set Device Name**
- **Machine name**: opnsense-router (or preferred name)
- Makes it easier to identify

**3. Configure ACLs (Managed via Git)**

ACLs are managed in the `tailnet/` submodule and automatically deployed via GitHub Actions. See `tailnet/policy.hujson` for the full policy.

Key ACL rules:
- **Admins** (`group:admin`) have full access to everything
- **All members** can access secure networks (192.168.0.0/20)
- **Tagged devices** can communicate with each other (full mesh)
- **Tailscale SSH** enabled for admins to all tagged devices

To make changes:
1. Edit `tailnet/policy.hujson`
2. Commit and push to GitHub
3. GitHub Actions deploys automatically

**4. Enable MagicDNS (Optional)**

**Navigation**: DNS → Settings

- ✓ Enable MagicDNS
- **Global nameservers**: 192.168.10.10 (Pi-hole)

This allows you to access devices by name (e.g., `ssh opnsense-router`)

---

## Part 6: Testing and Verification

### Test from OPNsense

```bash
# SSH into OPNsense
ssh root@192.168.1.1

# Check Tailscale status
tailscale status

# Expected output:
# 100.x.x.x   opnsense-router   you@   linux   active; relay "xxx"; subnet router
# 100.y.y.y   your-laptop       you@   linux   -

# Ping another Tailscale device
tailscale ping your-laptop

# Check advertised routes
tailscale status --json | grep routes
```

### Test from Remote Device

From any device on your Tailscale network (laptop, phone, etc.):

**Test network connectivity:**
```bash
# MGMT network
ping 192.168.1.1    # OPNsense router
ping 192.168.1.2    # Switch

# SERVERS network
ping 192.168.10.1   # Gateway
ping 192.168.10.10  # Pi-hole (primary)
ping 192.168.10.11  # Pi-hole (backup)

# WIFI_SECURE network
ping 192.168.11.1   # Gateway

# GUEST network
ping 192.168.20.1   # Gateway
```

**Test SSH access:**
```bash
ssh user@192.168.10.50  # SSH to a server on VLAN 10
```

**Test web access:**
```
http://192.168.10.10/admin  # Access Pi-hole
https://192.168.1.1         # Access OPNsense (if MGMT VLAN allows)
```

### Check Tailscale Network Status

**On remote device:**
```bash
tailscale status

# Should show opnsense-router as:
# 100.x.x.x   opnsense-router   you@   linux   active; subnet router
```

**Verify subnet routes:**
```bash
tailscale status --json | jq .Peer[].PrimaryRoutes

# Should show: ["192.168.0.0/16"]
```

---

## Part 7: Make Tailscale Start on Boot

### Configure Startup Script

Tailscale should start automatically with the FreeBSD service we configured earlier.

**Verify service is enabled:**
```bash
sysrc tailscaled_enable
# Should show: tailscaled_enable: YES
```

**If not enabled:**
```bash
sysrc tailscaled_enable="YES"
```

### Configure Tailscale to Start with Routes

Create a startup script that ensures Tailscale starts with proper configuration:

```bash
cat > /usr/local/etc/rc.d/tailscale-configure << 'EOF'
#!/bin/sh
# PROVIDE: tailscale_configure
# REQUIRE: tailscaled
# BEFORE: NETWORKING

. /etc/rc.subr

name="tailscale_configure"
start_cmd="tailscale_configure_start"
stop_cmd=":"

tailscale_configure_start() {
    sleep 5  # Wait for tailscaled to start
    /usr/local/bin/tailscale up --advertise-routes=192.168.0.0/16 --accept-routes
}

load_rc_config $name
run_rc_command "$1"
EOF

chmod +x /usr/local/etc/rc.d/tailscale-configure
sysrc tailscale_configure_enable="YES"
```

**Test startup:**
```bash
service tailscale_configure start
```

### Verify After Reboot

```bash
# Reboot OPNsense
reboot

# After reboot, SSH in and check
tailscale status

# Should show active with subnet routes
```

---

## Part 8: Advanced Configuration

### Exit Node Configuration (Optional)

If you want to route ALL traffic (including internet) through your OPNsense:

```bash
tailscale up --advertise-routes=192.168.0.0/16 --advertise-exit-node --accept-routes
```

**Then in Tailscale admin console:**
- Go to your OPNsense device
- Enable **Exit node**

**On client devices:**
- Select OPNsense as exit node
- All internet traffic will route through your home network

### Split DNS Configuration

**In Tailscale admin console:**

**Navigation**: DNS → Nameservers

**Add nameserver:**
- **Nameserver**: 192.168.10.10 (Pi-hole)
- **Restrict to subnet**: 192.168.0.0/16

This makes Pi-hole the DNS server for all queries to your home network subnets.

### Monitor Tailscale Traffic

**Check bandwidth usage:**
```bash
# On OPNsense
tailscale status --json | jq .Peer[].TxBytes
tailscale status --json | jq .Peer[].RxBytes
```

**OPNsense traffic graphs:**
- **Navigation**: Reporting → Traffic Graph
- Select **tailscale0** interface to see Tailscale bandwidth usage

---

## Part 9: Security Considerations

### Tailscale ACLs

Control which Tailscale users/devices can access your subnet:

**Example ACL (in Tailscale admin console):**
```json
{
  "acls": [
    // Allow only specific users to access home network
    {
      "action": "accept",
      "src": ["user1@example.com", "user2@example.com"],
      "dst": ["192.168.0.0/16:*"]
    },
    // Block everyone else
    {
      "action": "deny",
      "src": ["*"],
      "dst": ["192.168.0.0/16:*"]
    }
  ]
}
```

### OPNsense Firewall Rules

Consider adding rules to limit Tailscale access to specific VLANs:

**Example: Allow Tailscale to MGMT and SERVERS only**

**Navigation**: Firewall → Rules → Floating

**Modify existing Tailscale rule:**
- **Source**: Tailscale_Net
- **Destination**: MGMT net (192.168.1.0/24)
- **Destination**: SERVERS net (192.168.10.0/24)

**Block Tailscale from other networks:**
- **Action**: Block
- **Source**: Tailscale_Net
- **Destination**: WIFI_SECURE net, GUEST net

### Disable Tailscale SSH (Optional)

If you don't want Tailscale's SSH feature:
```bash
tailscale up --advertise-routes=192.168.0.0/16 --accept-routes --ssh=false
```

---

## Part 10: Troubleshooting

### Tailscale Not Starting on Boot

**Check service status:**
```bash
service tailscaled status
# Should show: tailscaled is running
```

**Start manually if needed:**
```bash
service tailscaled start
tailscale up --advertise-routes=192.168.0.0/16 --accept-routes
```

**Check logs:**
```bash
tail -f /var/log/tailscaled.log
```

### Cannot Access Subnets from Remote

**Check route approval:**
- Go to Tailscale admin console
- Verify subnet routes are approved

**Check firewall rules:**
- Navigation: Firewall → Rules → Floating
- Verify Tailscale_Net alias is correct (100.64.0.0/10)
- Verify rule is set to **Pass** and **Quick** is checked

**Test connectivity from OPNsense:**
```bash
# Can OPNsense reach the subnets?
ping 192.168.1.2    # Switch
ping 192.168.10.10  # Pi-hole
```

**Check IP forwarding:**
```bash
sysctl net.inet.ip.forwarding
# Should be: 1
```

### High Latency on Tailscale Connection

**Check relay usage:**
```bash
tailscale status

# Look for "relay" in status
# If using relay, direct connection failed
```

**Force direct connection:**
- Ensure UDP port 41641 is open on WAN
- Check if CGNAT is blocking P2P connections
- Consider port forwarding UDP 41641 on your ISP router

**Use DERP server closer to you:**
- Navigation: Tailscale admin console → Settings → DERP
- Select preferred region

### Authentication Expired

**Re-authenticate:**
```bash
tailscale up --advertise-routes=192.168.0.0/16 --accept-routes
# Follow authentication link
```

**Disable key expiry (prevent future expirations):**
- Tailscale admin console → Machines → [Your device]
- Click **Disable key expiry**

---

## Configuration Summary

### Tailscale Configuration
```bash
# On OPNsense:
tailscale up --advertise-routes=192.168.0.0/16 --accept-routes
```

### Firewall Rules Created
1. **WAN**: Allow UDP 41641 (Tailscale)
2. **Floating**: Allow Tailscale_Net (100.64.0.0/10) to all VLANs

### Service Configuration
```bash
sysrc tailscaled_enable="YES"
sysrc tailscale_configure_enable="YES"
```

---

## Part 11: Enable Remote Firewall Management via Tailscale

If you followed the [07_SECURITY_HARDENING.md](07_SECURITY_HARDENING.md) guide, you created a `Management_Access` alias that restricts web UI and SSH access to the MGMT network. To enable remote management via Tailscale, add the `Tailscale_Net` alias to this group.

### Update Management_Access Alias

**Navigation**: Firewall → Aliases

**Edit** the `Management_Access` alias:

- **Type**: Network group
- **Content**:
  ```
  MGMT__NET
  Tailscale_Net
  ```
- **Description**: `Networks allowed to access firewall management (MGMT + Tailscale)`

**Click:** Save → Apply Changes

**Note:** Using nested aliases keeps the configuration clean and automatically updates if the underlying network definitions change.

### Verify Remote Access

From a device connected to Tailscale (not on local network):

```bash
# Test web UI access
curl -k https://192.168.1.1

# Test SSH access (if enabled)
ssh admin@192.168.1.1
```

Both should now work from Tailscale-connected devices.

---

## Part 12: Tag Devices in Tailscale Admin Console

The Tailscale ACL policy (managed in `tailnet/policy.hujson`) uses tags for device-based access control. After adding devices to Tailscale, assign appropriate tags.

### Available Tags

| Tag | Purpose | Assign To |
|-----|---------|-----------|
| `tag:network` | Network infrastructure | OPNsense router |
| `tag:server` | General servers | Pi-holes, other servers |
| `tag:nas` | NAS/backup targets | NAS01, other NAS devices |
| `tag:mgmt-admin` | Firewall management access | Admin workstations |
| `tag:backup-client` | rsync backup clients | Devices that backup to NAS |
| `tag:dev` | Development devices | Dev machines |
| `tag:prod` | Production services | Production servers |

### How to Tag Devices

1. Go to https://login.tailscale.com/admin/machines
2. Click on the device
3. Click **Edit tags**
4. Add the appropriate tag(s)
5. Click **Save**

### Recommended Device Tags

| Device | Tags |
|--------|------|
| OPNsense Router | `tag:network` |
| Pi-hole Primary (RPi4) | `tag:server` |
| Pi-hole Backup (RPi3) | `tag:server` |
| NAS01 | `tag:nas`, `tag:server` |
| Admin Laptop | `tag:mgmt-admin`, `tag:backup-client` |
| Other Workstations | `tag:backup-client` |

### Access Control Summary

With the current ACL policy:
- **Admins** (group:admin) have full access to everything
- **All members** can access secure networks (MGMT, SERVERS, WIFI_SECURE via 192.168.0.0/20)
- **All tagged devices** can communicate with each other (full mesh)
- **All tagged devices** can access subnet routes (192.168.0.0/20)
- **tag:mgmt-admin** devices can SSH to network devices and access firewall management
- **Tailscale SSH**: Admins can SSH to any tagged device; tagged devices can SSH to each other

---

## Part 13: Centralized Logging via Tailscale

All devices on the Tailscale network can forward logs to the centralized syslog server for security monitoring and analysis.

### Syslog Server

| Setting | Value |
|---------|-------|
| **DNS Alias** | `syslog-server` (configured in Pi-hole) |
| **Actual Host** | `sands-log01` on Tailscale network |
| **Port** | `514` |
| **Protocol** | TCP (recommended) |

The `syslog-server` DNS alias is configured in Pi-hole's Local DNS Records (see [04_PIHOLE_SETUP.md](04_PIHOLE_SETUP.md#5-configure-local-dns-records)). This provides a single point of configuration — if the syslog server changes, update the Pi-hole DNS record instead of every device.

### Why Centralized Logging?

- **Security monitoring**: Detect brute force attacks, unauthorized access attempts
- **Troubleshooting**: Correlate events across multiple devices
- **Compliance**: Retain logs for audit purposes
- **Alerting**: Set up notifications for critical events

### Configure Logging on Linux Devices

Most Linux devices (Pi-holes, servers, workstations) use rsyslog. Create a configuration to forward logs over Tailscale.

#### Basic Setup: `/etc/rsyslog.d/99-remote.conf`

```bash
sudo tee /etc/rsyslog.d/99-remote.conf << 'EOF'
# Forward important logs to centralized syslog server via Tailscale
# Use @@ for TCP (reliable) or @ for UDP (faster)

# All logs (comprehensive)
# *.* @@syslog-server:514

# Or be selective (recommended):
# Authentication events (SSH, sudo, login)
auth,authpriv.*   @@syslog-server:514

# Kernel messages (security events, hardware)
kern.*            @@syslog-server:514

# Cron jobs (detect unauthorized tasks)
cron.*            @@syslog-server:514

# System daemon messages
daemon.*          @@syslog-server:514
EOF

# Restart rsyslog
sudo systemctl restart rsyslog
```

#### Verify Connectivity

```bash
# Test DNS resolution
dig +short syslog-server

# Test Tailscale connection to syslog server
tailscale ping sands-log01

# Generate a test log entry
logger -t test-syslog "Test message from $(hostname)"

# Check rsyslog status
sudo systemctl status rsyslog
```

### Device-Specific Logging Guides

| Device | Logging Guide |
|--------|---------------|
| Pi-hole (Primary/Backup) | [04_PIHOLE_SETUP.md](04_PIHOLE_SETUP.md#forwarding-logs-to-a-remote-syslog-server) |
| OPNsense | See below |
| NAS | Varies by NAS OS |

### OPNsense Syslog Configuration

OPNsense can forward firewall logs to the centralized syslog server.

**Navigation**: System → Settings → Logging / targets

**Add new target:**
- **Transport**: TCP(4)
- **Applications**: (leave empty for all, or select specific)
- **Levels**: Warning and above (or as needed)
- **Facilities**: (leave empty for all)
- **Hostname**: `syslog-server`
- **Port**: `514`
- **Description**: Centralized Syslog Server

**Click:** Save → Apply

**Note:** OPNsense uses Pi-hole for DNS, so `syslog-server` resolves correctly.

**Important logs to forward from OPNsense:**
- Firewall blocks (security monitoring)
- Authentication events (admin logins)
- System events (reboots, updates)
- VPN connections (Tailscale, OpenVPN)

### What to Log from Each Device

| Device Type | Priority Logs |
|-------------|---------------|
| **Pi-hole** | DNS queries, blocked domains, FTL service |
| **OPNsense** | Firewall blocks, auth events, VPN |
| **Servers** | auth, kern, daemon, application logs |
| **Workstations** | auth, kern (optional) |
| **NAS** | auth, file access (if supported) |

### Security Considerations

- **Encrypted transport**: Tailscale encrypts all traffic, so syslog over Tailscale is secure
- **No firewall changes needed**: Tailscale handles routing
- **MagicDNS**: Use hostnames for resilience if IPs change
- **Log retention**: Configure retention policy on syslog server

---

## Next Steps

After configuring Tailscale:
1. **Update Management_Access alias** (Part 11 above) to enable remote firewall management
2. **Tag devices** (Part 12 above) in Tailscale admin console
3. **Install Tailscale clients** on all devices you want to access remotely
4. **Test access** to each VLAN from remote devices
5. **Set up MagicDNS** for easier device access by hostname
6. **Enable HTTPS certificates** (see [09_TAILSCALE_HTTPS.md](09_TAILSCALE_HTTPS.md)) for secure web access
7. **Monitor usage** via OPNsense traffic graphs

---

## Related Documentation

- [02_VLAN_CONFIG.md](02_VLAN_CONFIG.md) - VLAN setup
- [05_OPNSENSE_PIHOLE_INTEGRATION.md](05_OPNSENSE_PIHOLE_INTEGRATION.md) - Pi-hole DNS
- [09_TAILSCALE_HTTPS.md](09_TAILSCALE_HTTPS.md) - Tailscale HTTPS certificates
- Official Tailscale docs: https://tailscale.com/kb/1019/subnets/
