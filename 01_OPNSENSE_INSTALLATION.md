# OPNsense Installation Guide

## Hardware Specifications

**Device**: Protectli FW41-0-4-32
- **Platform**: x86_64 fanless mini PC
- **CPU**: Intel Celeron J3160 (Quad-core, 1.6GHz)
- **RAM**: 4GB DDR3L (sufficient for OPNsense + VLANs + Tailscale)
- **Storage**: 32GB mSATA SSD
- **NICs** (4x Intel Gigabit Ethernet):
  - Intel I211 chipset (igb driver in FreeBSD)
  - `em0` - WAN (to ISP modem/router)
  - `em1` - VLAN trunk (to Aruba 2530-24G switch)
  - `em2` - Unused
  - `em3` - Break-glass emergency access (isolated 192.168.99.0/29, NOT in use for VLANs)
- **Power**: Low power consumption (~6W idle), fanless design
- **Form Factor**: Compact, ideal for always-on router duty

**NIC roles**: em1 carries all VLAN trunks (native VLAN 1/MGMT + tagged VLANs 10/11/20/21)
to the Aruba 2530-24G switch. em2 is unused. em3 is a physically isolated out-of-band port —
**configured immediately after first login**, before VLANs or firewall rules. It is your
safety net for all subsequent work.

---

## Download OPNsense

### Version: 26.1 "Witty Woodpecker" (Current)

> **Existing install?** Skip this section. See [13_OPNSENSE_26_1_UPGRADE.md](13_OPNSENSE_26_1_UPGRADE.md)
> for the in-place upgrade procedure.

**Download Image:**
```bash
cd ~/Downloads

# Download the amd64 DVD image
wget https://mirror.ams1.nl.leaseweb.net/opnsense/releases/26.1/OPNsense-26.1-dvd-amd64.iso.bz2

# Verify checksum (optional but recommended)
wget https://mirror.ams1.nl.leaseweb.net/opnsense/releases/26.1/OPNsense-26.1-checksums-amd64.sha256
sha256sum -c OPNsense-26.1-checksums-amd64.sha256 2>&1 | grep dvd

# Extract the image
bunzip2 OPNsense-26.1-dvd-amd64.iso.bz2
```

**Alternative Mirrors:**
- US: `https://mirror.wdc1.us.leaseweb.net/opnsense/releases/26.1/`
- EU: `https://mirror.fra10.de.leaseweb.net/opnsense/releases/26.1/`

---

## Create Bootable USB

### Find USB Device
```bash
lsblk
# Look for your USB drive (e.g., /dev/sdb)

# Verify it's the correct device
sudo fdisk -l /dev/sdX
```

### Write Image to USB
```bash
# CAUTION: This will erase all data on the USB drive!
# Replace /dev/sdX with your actual USB device

sudo dd if=OPNsense-26.1-dvd-amd64.iso of=/dev/sdX bs=4M status=progress oflag=sync

# Wait for completion, then safely eject
sudo sync
sudo eject /dev/sdX
```

### Alternative: Using Ventoy
If you have Ventoy installed on your USB:
```bash
cp OPNsense-26.1-dvd-amd64.iso /path/to/ventoy/usb/
```

---

## Installation Process

### 1. Boot from USB
1. Insert USB drive into router PC
2. Power on and enter BIOS/UEFI (usually F2, F12, or DEL)
3. Set USB as first boot device
4. Save and exit

### 2. OPNsense Installer
**Login credentials:**
- Username: `installer`
- Password: `opnsense`

**Installation Steps:**
1. Select: **Install (UFS)**
   - UFS is more reliable than ZFS on systems without ECC RAM

2. **Select target disk**: Choose your SSD (usually `ada0` or `da0`)

3. **Partitioning**: Choose **"Complete Disk"**

4. **Confirm installation**: Type `Yes` when prompted

5. **Installation progress**: Wait 5-10 minutes

6. **Set root password** — choose a strong password and record it securely

7. **Complete**: Select "Exit and Reboot"

8. **Remove USB drive** when prompted

---

## Initial Console Configuration

After first boot, you'll see the console menu.

### Assign Interfaces

**Option 1: Assign Interfaces**

```
Valid interfaces are:

em0    00:0d:b9:xx:xx:x0  (Intel I211)
em1    00:0d:b9:xx:xx:x1  (Intel I211)
em2    00:0d:b9:xx:xx:x2  (Intel I211)
em3    00:0d:b9:xx:xx:x3  (Intel I211)

Do you want to configure VLANs now? [y/n]: n

Enter the WAN interface name: em0
Enter the LAN interface name: em1

Do you want to proceed? [y/n]: y
```

> Only assign em0 (WAN) and em1 (LAN) at the console. em3 is configured via the web UI
> immediately after first login (see next section). em2 is not used.

### Set LAN IP Address

**Option 2: Set interface IP address**

```
Enter the number of the interface to configure: 2 (LAN)

Configure IPv4 address LAN interface via DHCP? [y/n]: n

Enter the new LAN IPv4 address: 192.168.1.1

Enter the new LAN IPv4 subnet bit count: 24

For a WAN, enter the new LAN IPv4 upstream gateway address: [press Enter for none]

Configure IPv6 address LAN interface via DHCP6? [y/n]: n

Do you want to enable the DHCP server on LAN? [y/n]: n
(We'll configure DHCP per-VLAN later via Dnsmasq)

Do you want to revert to HTTP as the web GUI protocol? [y/n]: n
(Keep HTTPS)
```

### First Access — via em1

Connect a laptop **directly to em1** (or through the Aruba switch if already cabled):

- Set laptop to static IP: `192.168.1.100 / 24`, gateway `192.168.1.1`
- Open browser: `https://192.168.1.1`
- Login: `root` / (password set during install)
- Accept the self-signed certificate warning

---

## Step 1 (IMMEDIATE): Configure em3 Break-Glass Port

> **Do this before configuring VLANs, firewall rules, bridging, or anything else.**
> em3 is your safety net. Once configured, you can perform all subsequent work knowing
> you have a physical out-of-band fallback if something goes wrong.

em3 is an isolated emergency access port on its own subnet. It is physically separate
from all other networks and provides web UI + SSH access to OPNsense only.

**Normal state**: cable unplugged. Connect only during emergencies.

### 1a — Assign em3

**Navigation**: Interfaces → Assignments

In the "New interface" dropdown at the bottom, select `em3`.
Set description: `MGMT_Only`
Click **+ Add** → **Save**

### 1b — Configure em3 IP

**Navigation**: Interfaces → [MGMT_Only]

| Setting | Value |
|---|---|
| Enable | ✓ |
| Description | `MGMT_Only` |
| IPv4 Configuration Type | Static IPv4 |
| IPv4 address | `192.168.99.1 / 29` |
| Block private networks | ☐ off |
| Block bogon networks | ☐ off |

Click **Save** → **Apply Changes**

Subnet:
```
Network:   192.168.99.0/29
Gateway:   192.168.99.1   ← OPNsense (web UI + SSH)
DHCP pool: 192.168.99.2 – 192.168.99.6   (5 leases max)
Broadcast: 192.168.99.7
```

### 1c — Enable DHCP on em3

**Navigation**: Services → Dnsmasq DNS & DHCP → DHCP ranges → **+**

| Setting | Value |
|---|---|
| Interface | MGMT_Only |
| Start | `192.168.99.2` |
| End | `192.168.99.6` |
| Gateway | `192.168.99.1` |
| Lease time | `3600` (1 hour) |

Click **Save** → **Apply**

### 1d — Firewall rules for em3

**Navigation**: Firewall → Rules → MGMT_Only

Add two Pass rules and nothing else (default deny handles the rest):

| Action | Protocol | Source | Destination | Port | Description |
|--------|----------|--------|-------------|------|-------------|
| Pass | TCP | MGMT_Only net | This Firewall | 443 | Break-glass HTTPS |
| Pass | TCP | MGMT_Only net | This Firewall | 22 | Break-glass SSH |

Click **Save** → **Apply Changes**

> Do NOT add internet access or routes to internal networks. em3 reaches OPNsense only.
> Do NOT add MGMT_Only to the `Management_Access` alias — these rules are separate from
> the Tailscale-only floating rule mandate.

### 1e — Verify

Plug a laptop into **em3**. It should:
1. Receive an IP in `192.168.99.2`–`.6` within seconds
2. Reach `https://192.168.99.1` (cert warning expected — this is the local IP, not the Tailscale hostname)
3. NOT be able to ping `192.168.1.1`, `192.168.10.x`, or any internet address

Once verified, **unplug the cable** and set it aside.

---

## Post-Installation Checklist

- [ ] OPNsense 26.1 installed and accessible via em1 (https://192.168.1.1)
- [ ] WAN (em0) is up and has internet connectivity
- [ ] LAN (em1) IP is 192.168.1.1 /24
- [ ] Root password recorded securely
- [ ] **em3 break-glass configured** (192.168.99.1/29, DHCP .2-.6, firewall rules)
- [ ] em3 verified — laptop gets IP and reaches https://192.168.99.1
- [ ] em3 cable unplugged (stored nearby for emergencies)

---

## Next Steps

With em3 break-glass in place, work through the docs in order:

### Access Setup (complete before VLANs or firewall rules)

2. **Tailscale** → [02_TAILSCALE_SETUP.md](02_TAILSCALE_SETUP.md)
   Install plugin, connect to tailnet, verify `opnsense.warthog-royal.ts.net` is reachable,
   enable Tailscale HTTPS cert, mandate Tailscale-only web UI access.
   After this step you have **two** out-of-band paths: em3 (physical) + Tailscale (remote).

### Network Configuration

3. **VLAN configuration** → [03_VLAN_CONFIG.md](03_VLAN_CONFIG.md)
   VLANs, DHCP ranges (em3 break-glass is already done — skip that phase)

4. **Switch configuration** → [04_SWITCH_CONFIG.md](04_SWITCH_CONFIG.md)
   Aruba 2530-24G switch setup

5. **Firewall rules** → [05_FIREWALL_RULES.md](05_FIREWALL_RULES.md)
   Aliases, floating rules, per-interface rules

6. **Pi-hole setup** → [06_PIHOLE_SETUP.md](06_PIHOLE_SETUP.md)
   OPNsense integration — Dnsmasq DHCP, DNS option, firewall alias

7. **UniFi AP setup** → [07_UNIFI_AP_SETUP.md](07_UNIFI_AP_SETUP.md)
   AP adoption and SSID-to-VLAN mapping

8. **Security hardening** → [08_SECURITY_HARDENING.md](08_SECURITY_HARDENING.md)
   Admin user, root SSH, remaining hardening steps

---

## Troubleshooting

See [TROUBLESHOOTING.md](TROUBLESHOOTING.md#opnsense) for common OPNsense issues.

---

## Backup Current OpenWRT Configuration

Before switching to OPNsense, backup your OpenWRT config:
```bash
./scripts/backup.sh
# This creates: backup/openwrt-backup-TIMESTAMP.tar.gz
```

Store this backup safely in case you need to revert.
