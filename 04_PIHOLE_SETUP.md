# Pi-hole Setup Guide

Complete setup guide for Pi-hole on Raspberry Pi, including pre-boot SD card configuration and Tailscale integration. This guide covers both primary and backup Pi-hole installation inline.

## Hardware Overview

| Role | Device | OS | IP Address | Switch Port | Tailscale Name |
|------|--------|-----|------------|-------------|----------------|
| **Primary** | Raspberry Pi 4 | RPi OS Lite (64-bit) | 192.168.10.10 | Port 2 → VLAN 10 | `pihole-primary` |
| **Backup** | Raspberry Pi 3 | RPi OS Lite (32-bit) | 192.168.10.11 | Port 3 → VLAN 10 | `pihole-backup` |

---

## Pre-Boot SD Card Configuration

After flashing with Raspberry Pi Imager, mount the boot partition and apply these configurations **before first boot**.

### 1. Flash the SD Card

Using Raspberry Pi Imager:

| Setting | Primary (RPi 4) | Backup (RPi 3) |
|---------|-----------------|----------------|
| **OS** | Raspberry Pi OS Lite (64-bit) | Raspberry Pi OS Lite (32-bit) |
| **Hostname** | `pihole-primary` | `pihole-backup` |
| **Enable SSH** | Yes | Yes |
| **Username/Password** | Your admin user | Your admin user |
| **Wireless LAN** | Skip (use Ethernet) | Skip (use Ethernet) |
| **Locale** | Your timezone | Your timezone |

### 2. Configure Static IP (Pre-Boot)

After flashing, the SD card will have two partitions:
- `bootfs` (FAT32) - Boot partition
- `rootfs` (ext4) - Root filesystem

**Mount the rootfs partition** and edit the network configuration:

```bash
# Mount the rootfs partition (adjust device as needed)
sudo mkdir /mnt
sudo mount /dev/mmcblk0p2 /mnt
sudo mount /dev/mmcblk0p1 /boot
```

**Disable BT and WIFI**:

```bash
# Mount the rootfs partition (adjust device as needed)
sudo tee -a /mnt/boot/config.txt << 'EOF'
dtoverlay=disable-wifi
dtoverlay=disable-bt
EOF
```

**For Primary Pi-hole (192.168.10.10):**
```bash
sudo tee -a /mnt/etc/dhcpcd.conf << 'EOF'

# Static IP configuration for primary Pi-hole
interface eth0
static ip_address=192.168.10.10/24
static routers=192.168.10.1
static domain_name_servers=1.1.1.1 8.8.8.8
EOF
```

**For Backup Pi-hole (192.168.10.11):**
```bash
sudo tee -a /mnt/etc/dhcpcd.conf << 'EOF'

# Static IP configuration for backup Pi-hole
interface eth0
static ip_address=192.168.10.11/24
static routers=192.168.10.1
static domain_name_servers=1.1.1.1 8.8.8.8
EOF
```

```bash
# Unmount when done
sudo umount /mnt/boot
sudo umount /mnt
```

### 4. First Boot

1. Insert SD card into Raspberry Pi
2. Connect Ethernet to appropriate switch port (see Hardware Overview)
3. Power on the Pi
4. Wait 2-3 minutes for first boot to complete
5. SSH to the Pi:

```bash
# Primary
ssh pihole-primary@192.168.10.10

# Backup
ssh pihole-backup@192.168.10.11
```

---

## Pi-hole Installation

Perform these steps on **both** primary and backup Pi-holes.

### 1. Update System

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Install Pi-hole

```bash
curl -sSL https://install.pi-hole.net | bash
```

During installation:
- **Upstream DNS**: Select Cloudflare (1.1.1.1) or your preference
- **Blocklists**: Use default StevenBlack list
- **Admin Interface**: Yes
- **Web Server**: Yes (lighttpd)
- **Log Queries**: Yes
- **Privacy Mode**: Show everything (for troubleshooting)

**Note the admin password** displayed at the end of installation.

### 3. Set Admin Password

Change the randomly-generated installation password:

```bash
# Interactive method (recommended - password not shown)
sudo pihole setpassword

# Non-interactive method
sudo pihole setpassword "YourSecurePassword123!"
```

**Important:** Use different admin passwords for primary and backup Pi-holes.

### 4. Set DNS Listening Mode

Pi-hole must listen on all interfaces for cross-VLAN and Tailscale DNS access:

```bash
sudo vi /etc/pihole/pihole.toml
```

Find and modify the `[dns]` section:
```toml
[dns]
  # Listen on all interfaces (required for cross-VLAN and Tailscale DNS)
  listeningMode = "ALL"
```

Restart Pi-hole:
```bash
sudo pihole reloaddns
```

---

## Tailscale Setup

Install Tailscale **immediately after Pi-hole** so you can access the web interface via Tailscale SSH for all remaining configuration. This minimizes the time you need to use IP-based SSH.

### 1. Install Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

### 2. Authenticate and Connect

```bash
sudo tailscale up --accept-routes --ssh
```

**Important flags:**
- `--accept-routes`: Accept subnet routes from OPNsense (if configured)
- `--accept-dns=false`: Don't use Tailscale DNS (Pi-hole IS the DNS server)
- `--ssh`: Enable Tailscale SSH for secure remote access

Follow the authentication URL to log in.

### 3. Rename in Tailscale Admin

After joining, rename the device in Tailscale admin for clarity:

1. Go to https://login.tailscale.com/admin/machines
2. Find the device (will show as `pihole-primary` or `pihole-backup`)
3. Click **Edit machine name** if needed to match:
   - Primary: `pihole-primary`
   - Backup: `pihole-backup`
4. Click **Edit tags** → Add tag: `server`
5. Click **Save**

### 4. Enable IP Forwarding

```bash
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### 5. Verify Tailscale Connection

```bash
# Get Tailscale IP
tailscale ip -4
# Example output: 100.x.x.x

# Test connectivity
tailscale status
```

### 6. Switch to Tailscale SSH

From now on, access Pi-hole via Tailscale SSH:

```bash
# Primary
tailscale ssh pihole-primary

# Backup
tailscale ssh pihole-backup
```

No more need for IP-based SSH access!

---

## Pi-hole Web Configuration

Access the Pi-hole admin interface via Tailscale:

| Pi-hole | URL |
|---------|-----|
| Primary | `http://pihole-primary.warthog-royal.ts.net/admin` |
| Backup | `http://pihole-backup.warthog-royal.ts.net/admin` |

Or use the local IP if on the same network: `http://192.168.10.10/admin` or `http://192.168.10.11/admin`

### 1. Verify DNS Listening Mode

1. Go to **Settings → DNS**
2. Under **Interface settings**, verify **"Permit all origins"** is selected
3. Click **Save** if changed

### 2. Configure Conditional Forwarding (Hostnames)

To see client hostnames instead of just IP addresses in Pi-hole logs, configure reverse DNS forwarding to OPNsense.

**Navigate to**: Settings → DNS → Advanced DNS settings

Scroll to **Reverse server (former conditional forwarding)** and add the following entries:

```
true,192.168.1.0/24,192.168.1.1
true,192.168.10.0/24,192.168.10.1
true,192.168.11.0/24,192.168.11.1
true,192.168.20.0/24,192.168.20.1
```

| Network | Entry |
|---------|-------|
| MGMT | `true,192.168.1.0/24,192.168.1.1` |
| SERVERS | `true,192.168.10.0/24,192.168.10.1` |
| WIFI_SECURE | `true,192.168.11.0/24,192.168.11.1` |
| GUEST | `true,192.168.20.0/24,192.168.20.1` |

**Format:** `<enabled>,<ip-address>/<prefix-len>,<server>[#<port>][,<domain>]`

Click **Save** to apply changes.

**Configure on both Pi-holes** for consistent hostname resolution.

### 3. Configure Local DNS Records

Pi-hole can serve as a local DNS resolver for infrastructure services.

**Navigate to**: Local DNS → DNS Records

Add the following records on **both** Pi-holes:

| Domain | IP Address | Purpose |
|--------|------------|---------|
| `syslog-server` | *(Tailscale IP of sands-log01)* | Centralized logging server |
| `pihole-primary` | `192.168.10.10` | Primary Pi-hole |
| `pihole-backup` | `192.168.10.11` | Backup Pi-hole |
| `opnsense` | `192.168.1.1` | OPNsense router |
| `nas01` | `192.168.10.20` | Primary NAS |

**To get the Tailscale IP for sands-log01:**
```bash
tailscale status | grep sands-log01
# Or use MagicDNS
dig +short sands-log01.warthog-royal.ts.net
```

---

## HTTPS via Tailscale

Enable HTTPS access to Pi-hole admin via Tailscale Serve:

```bash
sudo tailscale serve --bg http://localhost:80
```

Access via:
- Primary: `https://pihole-primary.warthog-royal.ts.net/admin`
- Backup: `https://pihole-backup.warthog-royal.ts.net/admin`

This provides automatic certificate provisioning and renewal.

For more options, see [09_TAILSCALE_HTTPS.md](09_TAILSCALE_HTTPS.md).

---

## Gravity Sync (Pi-hole Synchronization)

Keep blocklists and settings synchronized between primary and backup Pi-hole using **Gravity Sync**.

### Install Gravity Sync

On **both** Pi-holes:

```bash
curl -sSL https://raw.githubusercontent.com/vmstan/gs-install/main/gs-install.sh | bash
```

### Configure Gravity Sync on Backup Pi-hole

```bash
gravity-sync config
```

Enter when prompted:
- **Remote Host**: `192.168.10.10` (primary Pi-hole)
- **Remote User**: Your SSH username on primary Pi-hole
- **SSH Key**: Generate or use existing key

### Set Up SSH Key Authentication

On the backup Pi-hole:

```bash
# Generate SSH key if needed
ssh-keygen -t ed25519 -C "gravity-sync"

# Copy key to primary Pi-hole
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@192.168.10.10
```

### Test and Enable Sync

```bash
# Test sync (one-time pull from primary)
gravity-sync pull

# Verify blocklists match
sudo pihole -g

# Enable automatic sync (runs every 30 minutes)
gravity-sync automate
```

### What Gets Synced

| Synced | Not Synced (configure manually) |
|--------|--------------------------------|
| Blocklists (adlists) | Admin password |
| Whitelist/Blacklist | Network settings |
| Regex filters | Upstream DNS servers |
| Client groups | DHCP settings |
| Custom DNS records | |

---

## OPNsense Integration

### DHCP Configuration for Redundant DNS

Update OPNsense Dnsmasq to provide both Pi-holes as DNS servers via DHCP options.

**Navigate to**: Services → Dnsmasq DNS & DHCP → DHCP options

| Setting | Value |
|---------|-------|
| **Interface** | Any |
| **Type** | Set |
| **Option** | dns-server [6] |
| **Value** | 192.168.10.10,192.168.10.11 |
| **Description** | Pi-hole DNS servers (primary + backup) |

Click **Save** → **Apply**

### Update Firewall Alias

**Navigate to**: Firewall → Aliases

Edit **Pi_hole_DNS** alias:
```
192.168.10.10
192.168.10.11
```

Click **Save** → **Apply Changes**

The existing floating rule "Allow DNS to Pi-hole" will automatically allow DNS traffic to both Pi-holes.

See [05_OPNSENSE_PIHOLE_INTEGRATION.md](05_OPNSENSE_PIHOLE_INTEGRATION.md) for complete integration details.

---

## Tailscale DNS Configuration

### Option A: Use Pi-hole as Tailscale DNS (Recommended)

In the Tailscale admin console (https://login.tailscale.com/admin/dns):

1. Go to **DNS** settings
2. Add **Global nameserver**: Use the Tailscale IP of primary Pi-hole
3. Enable **Override local DNS**

Now all Tailscale devices use Pi-hole for DNS, even when remote.

### Option B: Access Pi-hole Only When Needed

Access Pi-hole via Tailscale hostname when connected to tailnet:
- Admin: `http://pihole-primary.warthog-royal.ts.net/admin`
- DNS: Configure devices to use Pi-hole's Tailscale IP

---

## Verification

### Test DNS Resolution

```bash
# Test primary
nslookup google.com 192.168.10.10

# Test backup
nslookup google.com 192.168.10.11

# Test ad blocking on both
nslookup ads.google.com 192.168.10.10
nslookup ads.google.com 192.168.10.11
# Should return 0.0.0.0 or NXDOMAIN
```

### Test Tailscale Access

```bash
# Test DNS via Tailscale
nslookup google.com pihole-primary.warthog-royal.ts.net

# Access admin interface
curl -I http://pihole-primary.warthog-royal.ts.net/admin
```

### Test Failover

1. Stop Pi-hole on primary: `sudo pihole disable`
2. From a client, verify DNS still works (should use backup)
3. Re-enable primary: `sudo pihole enable`

### Check Gravity Sync Status

```bash
# On backup Pi-hole
gravity-sync status
gravity-sync logs
```

---

## Maintenance

### Update Pi-hole

```bash
# On each Pi-hole
sudo pihole -up
```

### Update Blocklists

```bash
sudo pihole -g
```

### Backup Configuration

**Via Web Interface:**
- Go to **Settings → Teleporter → Export**

**Via Command Line:**
```bash
sudo pihole-FTL --teleporter /path/to/backup.zip
```

### View Statistics

```bash
sudo pihole -c
```

### Check Pi-hole Logs

```bash
# View real-time query log
sudo pihole -t

# Check for errors
sudo pihole -d
```

### Manual Gravity Sync

```bash
# On backup - pull latest from primary
gravity-sync pull

# Push backup config to primary (use with caution)
gravity-sync push
```

---

## Troubleshooting

See [TROUBLESHOOTING.md](TROUBLESHOOTING.md#pi-hole) for Pi-hole issues.

---

## Pi-hole Remote iOS App Setup

[Pi-hole Remote](https://apps.apple.com/us/app/pi-hole-remote/id1515445551) by RocketScience IT provides iOS management of your Pi-hole from anywhere on your Tailscale network.

### Prerequisites

- Pi-hole v6.0 or later
- Tailscale installed on your iPhone
- Pi-hole accessible via Tailscale

### Configure an App Password

Pi-hole v6 uses session-based authentication. For mobile apps, use an **App Password**:

1. Open Pi-hole admin via Tailscale
2. Go to **Settings → Web Interface/API**
3. Switch to **Expert** mode
4. Click **Configure app password**
5. Enable and copy the generated password

### Configure Pi-hole Remote App

| Setting | Primary | Backup |
|---------|---------|--------|
| **Host** | `pihole-primary.warthog-royal.ts.net` | `pihole-backup.warthog-royal.ts.net` |
| **Port** | `443` (with HTTPS) or `80` | `443` (with HTTPS) or `80` |
| **Password** | App Password | App Password |
| **Use HTTPS** | On (if using Tailscale Serve) | On (if using Tailscale Serve) |

Pi-hole Remote supports multiple instances - add both for combined statistics and control.

---

## Forwarding Logs to a Remote Syslog Server

Centralized logging allows you to aggregate Pi-hole DNS queries and system logs on a remote syslog server.

### Prerequisites

- Syslog server (rsyslog, syslog-ng, Graylog) on your network
- For this network: `sands-log01` on the Tailscale network

### Install rsyslog

```bash
sudo apt install rsyslog -y
sudo systemctl enable rsyslog
```

### Simple Configuration (All Logs)

Create `/etc/rsyslog.d/90-remote.conf`:

```bash
sudo tee /etc/rsyslog.d/90-remote.conf << 'EOF'
# Forward all logs to remote syslog server
*.* @@syslog-server:514
EOF
```

### Pi-hole Specific Configuration

For structured Pi-hole log forwarding, create `/etc/rsyslog.d/10-pihole.conf`:

```bash
sudo tee /etc/rsyslog.d/10-pihole.conf << 'EOF'
# Pi-hole log forwarding to remote syslog server
module(load="imfile")
module(load="mmnormalize")

# Pi-hole DNS Query Log
input(type="imfile"
      File="/var/log/pihole/pihole.log"
      Tag="pihole-dns:"
      Severity="info"
      Facility="local0"
      PersistStateInterval="100"
      reopenOnTruncate="on"
      readMode="2"
      ruleset="rs_pihole_dns")

template(name="T_PiHoleDNS" type="string"
         string="%$!date% %hostname% pihole-dns[%$!pid%]: %$!msg%\n")

ruleset(name="rs_pihole_dns") {
    action(type="mmnormalize"
           ruleBase="/etc/rsyslog.d/pihole-dns.rulebase")

    action(type="omfwd"
           target="syslog-server"
           port="514"
           protocol="tcp"
           template="T_PiHoleDNS")
}

# Pi-hole FTL Service Log
input(type="imfile"
      File="/var/log/pihole/FTL.log"
      Tag="pihole-ftl:"
      Severity="info"
      Facility="local1"
      PersistStateInterval="100"
      reopenOnTruncate="on"
      readMode="2"
      ruleset="rs_pihole_ftl")

template(name="T_PiHoleFTL" type="string"
         string="%$!date%T%$!time%.%$!ms% %$!tz% %hostname% pihole-ftl[%$!pid%]: %$!level%: %$!msg%\n")

ruleset(name="rs_pihole_ftl") {
    action(type="mmnormalize"
           ruleBase="/etc/rsyslog.d/pihole-ftl.rulebase")

    action(type="omfwd"
           target="syslog-server"
           port="514"
           protocol="tcp"
           template="T_PiHoleFTL")
}
EOF
```

Create rulebase files:

```bash
# DNS log rulebase
sudo tee /etc/rsyslog.d/pihole-dns.rulebase << 'EOF'
rule=:%date:date-rfc3164% %program:char-to:[%[%pid:char-to:]%]: %msg:rest%
EOF

# FTL log rulebase
sudo tee /etc/rsyslog.d/pihole-ftl.rulebase << 'EOF'
rule=:%date:date-iso% %time:time-24hr%.%ms:number% %tz:word% [%pid:number%/T%tid:number%] %level:char-to::%: %msg:rest%
rule=:%date:date-iso% %time:time-24hr%.%ms:number% %tz:word% [%pid:number%M] %level:char-to::%: %msg:rest%
EOF
```

### Security Logs

```bash
sudo tee /etc/rsyslog.d/20-security.conf << 'EOF'
# Forward authentication and security-related logs
auth,authpriv.*   @@syslog-server:514
kern.*            @@syslog-server:514
cron.*            @@syslog-server:514
EOF
```

### Apply Configuration

```bash
sudo rsyslogd -N1           # Validate
sudo systemctl restart rsyslog
sudo systemctl status rsyslog
```

### Verify Log Forwarding

```bash
# Generate test entry
logger -t pihole-test "Test message from Pi-hole"

# Verify Tailscale connectivity to syslog server
tailscale ping sands-log01
```

**Apply the same rsyslog configuration to both Pi-holes** for complete DNS query logging during failover.

---

## Security Notes

- Use different admin passwords for primary and backup Pi-holes
- Use App Passwords for third-party apps instead of admin password
- Tailscale provides encrypted access - no additional VPN needed
- Keep Pi-hole and Tailscale updated regularly
- Review query logs periodically for unusual patterns
- Consider enabling HTTPS via Tailscale Serve for admin interface
