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
  - `em1` - LAN/MGMT (VLAN trunk to managed switch — carries all VLANs)
  - `em2` - Available for future use
  - `em3` - Available for future use
- **Power**: Low power consumption (~6W idle), fanless design
- **Form Factor**: Compact, ideal for always-on router duty

**Note**: Intel NICs provide excellent performance and reliability for VLAN tagging and routing. All networks (MGMT, SERVERS, WIFI_SECURE, GUEST, HomeAssist) trunk through em1 to the managed switch using 802.1Q VLAN tags. em2 and em3 are unused.

## Download OPNsense

### Version: 25.7 (Latest Stable)

**Download Image:**
```bash
cd ~/Downloads

# Download the amd64 DVD image
wget https://mirror.ams1.nl.leaseweb.net/opnsense/releases/25.7/OPNsense-25.7-dvd-amd64.iso.bz2

# Verify checksum (optional but recommended)
wget https://mirror.ams1.nl.leaseweb.net/opnsense/releases/25.7/OPNsense-25.7-checksums-amd64.sha256
sha256sum -c OPNsense-25.7-checksums-amd64.sha256 2>&1 | grep dvd

# Extract the image
bunzip2 OPNsense-25.7-dvd-amd64.iso.bz2
```

**Alternative Mirrors:**
- US: `https://mirror.wdc1.us.leaseweb.net/opnsense/releases/25.7/`
- EU: `https://mirror.fra10.de.leaseweb.net/opnsense/releases/25.7/`

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

sudo dd if=OPNsense-25.7-dvd-amd64.iso of=/dev/sdX bs=4M status=progress oflag=sync

# Wait for completion, then safely eject
sudo sync
sudo eject /dev/sdX
```

### Alternative: Using Ventoy
If you have Ventoy installed on your USB:
```bash
# Simply copy the ISO to the Ventoy USB drive
cp OPNsense-25.7-dvd-amd64.iso /path/to/ventoy/usb/
```

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
   - ZFS is only recommended if you have ECC RAM

2. **Select target disk**: Choose your SSD (usually ada0 or da0)

3. **Partitioning**: Choose **"Complete Disk"**
   - Uses entire disk for OPNsense
   - Automatically creates swap partition

4. **Confirm installation**: Type `Yes` when prompted

5. **Installation progress**: Wait 5-10 minutes

6. **Set root password**:
   - Choose a strong password
   - Write it down securely

7. **Complete**: Select "Exit and Reboot"

8. **Remove USB drive** when prompted

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

**Note**: We only assign em0 (WAN) and em1 (LAN/trunk) at this stage. All additional networks are created as VLAN sub-interfaces on em1 via the web UI — see [02_VLAN_CONFIG.md](02_VLAN_CONFIG.md).

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
(We'll configure DHCP per-VLAN later)

Do you want to revert to HTTP as the web GUI protocol? [y/n]: n
(Keep HTTPS for security)
```

### Access Web Interface

1. Connect your PC to the router's LAN port (em1)
2. Set your PC to DHCP or use static IP: 192.168.1.100/24
3. Open browser: `https://192.168.1.1`
4. Login:
   - Username: `root`
   - Password: (the password you set during installation)

**Note**: Your browser will warn about the self-signed certificate. This is normal - click "Advanced" and proceed.

## Post-Installation Checklist

- [ ] System accessible via web interface at https://192.168.1.1
- [ ] WAN interface (em0) is up and has connectivity
- [ ] LAN interface (em1) has IP 192.168.1.1
- [ ] Root password is documented securely
- [ ] Browser bookmarked for https://192.168.1.1

## Next Steps

After completing installation:
1. **IMPORTANT**: See [07_SECURITY_HARDENING.md](07_SECURITY_HARDENING.md) to create admin user and harden security
2. See [02_VLAN_CONFIG.md](02_VLAN_CONFIG.md) for VLAN configuration
3. See [05_OPNSENSE_PIHOLE_INTEGRATION.md](05_OPNSENSE_PIHOLE_INTEGRATION.md) for Pi-hole setup

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
