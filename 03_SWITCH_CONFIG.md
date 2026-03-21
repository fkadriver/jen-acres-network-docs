# NetGear GS310TP Switch Configuration

Complete configuration guide for NetGear GS310TP managed switch to support the VLAN architecture with UniFi U6-Pro access points.

## Architecture Overview

The switch handles all VLAN trunking between the router and network devices:
- **Router trunk (Port 1)**: Carries all VLANs (1, 10, 11, 20, 21)
- **Server ports (Ports 2-5)**: Access ports on VLAN 10
- **AP trunk ports (Ports 6-7)**: Carry wireless VLANs (11, 20, 21) to UniFi U6-Pro APs
- **Management port (Port 8)**: Access port on VLAN 1

## Quick Reference - Netgear Menu Paths

| Task | Menu Path |
|------|-----------|
| Create VLANs | `VLAN → 802.1Q → Advanced → VLAN Configuration` |
| Set Port PVID | `VLAN → 802.1Q → Advanced → Port PVID Configuration` |
| Configure Membership | `VLAN → 802.1Q → Advanced → VLAN Membership` |
| View Configuration | `VLAN → 802.1Q → Advanced → VLAN Status` |

## Key Terminology

- **T** (Tagged) = Trunk port - carries multiple VLANs with VLAN tags
- **U** (Untagged) = Access port - carries one VLAN without tags
- **PVID** = Port VLAN ID - default VLAN for untagged traffic on that port
- **-** = Port is not a member of this VLAN

## Configuration Workflow Summary

1. **Access switch** and change admin password
2. **Set static IP** to 192.168.1.2
3. **Create VLANs** (10, 11, 20, 21 - VLAN 1 exists by default)
4. **Configure Router Port** (Port 1) - Trunk with VLAN 1 (native) and VLANs 10, 11, 20, 21 (tagged)
5. **Configure Server Ports** (Ports 2-5) - Access ports on VLAN 10
6. **Configure AP Ports** (Ports 6-7) - Trunk with VLAN 11 (native) and VLANs 20, 21 (tagged)
7. **Configure Management Port** (Port 8) - Access port on VLAN 1
8. **Set PVIDs** - Match PVID to untagged/native VLAN
9. **Test Connectivity** - Verify all VLANs work

## Network Design

| VLAN ID | Network | Purpose | Switch Ports | Notes |
|---------|---------|---------|--------------|-------|
| 1 | 192.168.1.0/24 | Management (MGMT) | Port 1 (trunk), Port 8 (access) | Switch IP: 192.168.1.2 |
| 10 | 192.168.10.0/24 | Servers | Port 1 (trunk), Ports 2-5 (access) | Pi-hole Primary: .10, Backup: .11 |
| 11 | 192.168.11.0/24 | WiFi Secure | Port 1 (trunk), Ports 6-7 (trunk) | Secure wireless SSID |
| 20 | 192.168.20.0/24 | Guest | Port 1 (trunk), Ports 6-7 (trunk) | Guest wireless SSID |
| 21 | 192.168.21.0/24 | HomeAssist | Port 1 (trunk), Ports 6-7 (trunk) | IoT wireless SSID |

## Step-by-Step Configuration

### 1. Access Switch Management Interface

**Default IP:** `192.168.0.239` (check switch label for current defaults)

**Web Interface:**
- Open browser to: `http://192.168.0.239` or `https://192.168.0.239`
- Login with default credentials (usually `admin` / `password` - check switch label)
- **Immediately change the admin password**

**Set Static IP Address:**

After logging in, configure the switch with a static IP on the management VLAN:

1. Go to: **System → Management → IP Configuration**
2. Set configuration mode to **Static**
3. Configure:
   - **IP Address:** `192.168.1.2`
   - **Subnet Mask:** `255.255.255.0`
   - **Default Gateway:** `192.168.1.1` (OPNsense router)
4. Click **Apply**
5. Switch will become accessible at `http://192.168.1.2`

**Note:** After applying, reconnect to the switch at the new IP address: `http://192.168.1.2`

### 2. Create VLANs

**Menu Path:** `VLAN → 802.1Q → Advanced → VLAN Configuration`

**Create these VLANs:**

1. **VLAN 1** - MGMT (already exists by default, no action needed)

2. **VLAN 10** - Servers
   - Click "Add"
   - VLAN ID: `10`
   - VLAN Name: `Servers` (optional)
   - Click Apply

3. **VLAN 11** - WiFi Secure
   - Click "Add"
   - VLAN ID: `11`
   - VLAN Name: `WiFi_Secure` (optional)
   - Click Apply

4. **VLAN 20** - Guest
   - Click "Add"
   - VLAN ID: `20`
   - VLAN Name: `Guest` (optional)
   - Click Apply

5. **VLAN 21** - HomeAssist
   - Click "Add"
   - VLAN ID: `21`
   - VLAN Name: `HomeAuto` (optional)
   - Click Apply

**Verify:** Go to `VLAN → 802.1Q → Advanced → VLAN Configuration` and confirm VLANs 1, 10, 11, 20, and 21 are listed.

### 3. Configure Trunk Port to OPNsense Router

The port connected to your router needs to carry traffic for all VLANs.

**Configuration Goal:**
- **Port**: Connected to OPNsense em1 (Port 1 on switch)
- **Native VLAN**: 1 (untagged for MGMT)
- **Tagged VLANs**: 10, 11, 20, 21
- **PVID**: 1 (default VLAN)
- **Purpose**: Carries all network traffic between switch and router

**Configure VLAN Membership:**

1. Go to: **VLAN → 802.1Q → Advanced → VLAN Membership**
2. For **VLAN ID 1**:
   - Find Port 1 (router port)
   - Set to **U** (Untagged) - this is the native VLAN
3. For **VLAN ID 10**:
   - Find Port 1
   - Set to **T** (Tagged)
4. For **VLAN ID 11**:
   - Find Port 1
   - Set to **T** (Tagged)
5. For **VLAN ID 20**:
   - Find Port 1
   - Set to **T** (Tagged)
6. For **VLAN ID 21**:
   - Find Port 1
   - Set to **T** (Tagged)

**Configure Port PVID:**

1. Go to: **VLAN → 802.1Q → Advanced → Port PVID Configuration**
2. Set Port 1 PVID to **1** (management VLAN)
3. Click Apply

**Verification:**
- Port 1 should show **U** for VLAN 1, **T** for VLANs 10, 11, 20, 21
- Port 1 PVID should be 1

### 4. Configure Access Ports

Access ports connect end devices to specific VLANs (untagged).

| Port | Device Type | VLAN | Mode | Config |
|------|-------------|------|------|--------|
| 1 | OPNsense Router (em1) | 1, 10, 11, 20, 21 | Trunk | Untagged: VLAN 1, Tagged: 10, 11, 20, 21 |
| 2 | Pi-hole Primary (RPi4) | 10 | Access | Untagged VLAN 10, PVID 10 |
| 3 | Pi-hole Backup (RPi3) | 10 | Access | Untagged VLAN 10, PVID 10 |
| 4 | NAS01 | 10 | Access | Untagged VLAN 10, PVID 10 |
| 5 | VM01 | 10 | Access | Untagged VLAN 10, PVID 10 |
| 6 | UniFi U6-Pro #1 | 11, 20, 21 | Trunk | Untagged: VLAN 11, Tagged: 20, 21 |
| 7 | UniFi U6-Pro #2 | 11, 20, 21 | Trunk | Untagged: VLAN 11, Tagged: 20, 21 |
| 8 | Management | 1 | Access | Untagged VLAN 1, PVID 1 |

**For each server port (Ports 2-5):**

1. **Set Port PVID:**
   - Go to: **VLAN → 802.1Q → Advanced → Port PVID Configuration**
   - Find Port 2 (or 3, 4, 5)
   - Set PVID to **10**
   - Click Apply

2. **Configure VLAN Membership:**
   - Go to: **VLAN → 802.1Q → Advanced → VLAN Membership**
   - Select **VLAN ID 10**
   - Find Port 2, set to **U** (Untagged)
   - Click Apply
   - Go back and select **VLAN ID 1**
   - Find Port 2, ensure it's **NOT** a member (set to **-**)
   - Click Apply
   - Repeat for Ports 3, 4, and 5

**For management port (Port 8):**
- PVID = 1, Untagged on VLAN 1 (default configuration)

**Important:** Each access port should only be an **untagged** member of ONE VLAN, with PVID matching that VLAN.

### 5. Configure AP Trunk Ports (Ports 6-7)

UniFi U6-Pro access points require trunk ports to receive multiple VLANs for different SSIDs.

**Configuration Goal:**
- **Ports**: 6 and 7 (connected to UniFi U6-Pro APs)
- **Native VLAN**: 11 (WIFI_SECURE - management traffic for APs)
- **Tagged VLANs**: 20, 21 (for Guest and HomeAssist SSIDs)
- **PVID**: 11
- **PoE**: Enabled (802.3at PoE+ required)

**For each AP port (Ports 6-7):**

1. **Set Port PVID:**
   - Go to: **VLAN → 802.1Q → Advanced → Port PVID Configuration**
   - Find Port 6 (or 7)
   - Set PVID to **11**
   - Click Apply

2. **Configure VLAN Membership:**
   - Go to: **VLAN → 802.1Q → Advanced → VLAN Membership**
   - Select **VLAN ID 11**:
     - Find Port 6, set to **U** (Untagged) - native VLAN for AP management
   - Select **VLAN ID 20**:
     - Find Port 6, set to **T** (Tagged)
   - Select **VLAN ID 21**:
     - Find Port 6, set to **T** (Tagged)
   - Ensure Port 6 is **NOT** a member of VLANs 1 and 10 (set to **-**)
   - Click Apply
   - Repeat for Port 7

3. **Verify PoE is enabled:**
   - Go to: **System → PoE → PoE Port Configuration**
   - Ensure Ports 6 and 7 show:
     - Admin Mode: **Enabled**
     - Power Mode: **802.3at** (or Auto)

**Note:** The UniFi U6-Pro requires 802.3at (PoE+) which provides up to 30W per port. The GS310TP supports this standard.

### 6. Configuration Summary

This table mirrors the NetGear **PVID Configuration** page layout:

| Interface | Device | PVID | VLAN Member | Untagged VLANs | Tagged VLANs |
|-----------|--------|------|-------------|----------------|--------------|
| g1 | Router (em1) | 1 | 1,10-11,20-21 | 1 | 10-11,20-21 |
| g2 | Pi-hole Primary | 10 | 1,10 | 10 | 1 |
| g3 | Pi-hole Backup | 10 | 1,10 | 10 | 1 |
| g4 | NAS01 | 10 | 1,10 | 10 | 1 |
| g5 | VM01 | 10 | 1,10 | 10 | 1 |
| g6 | U6-Pro #1 | 11 | 1,11,20-21 | 11 | 1,20-21 |
| g7 | U6-Pro #2 | 11 | 1,11,20-21 | 11 | 1,20-21 |
| g8 | Management | 1 | 1 | 1 | None |
| g9 | (SFP - unused) | 1 | 1 | 1 | None |
| g10 | (SFP - unused) | 1 | 1 | 1 | None |

**Note:** Ports g9 and g10 are SFP (fiber) ports, not currently used.

**Column Reference:**
- **PVID**: Port VLAN ID - default VLAN for untagged incoming traffic
- **VLAN Member**: All VLANs the port participates in
- **Untagged VLANs**: Traffic exits port without VLAN tag (access mode)
- **Tagged VLANs**: Traffic exits port with VLAN tag (trunk mode)

## Example Port Assignment

```
┌─────────────────────────────────────────────────────────┐
│              NetGear GS310TP Switch (PoE+)              │
├─────────────────────────────────────────────────────────┤
│ Port 1:  Trunk to OPNsense router (em1)                 │
│          Native VLAN 1 (U), Tagged: 10,11,20,21 (T)     │
├─────────────────────────────────────────────────────────┤
│ Port 2:  Pi-hole Primary RPi4 (192.168.10.10)           │
│          Access, Untagged VLAN 10, PVID 10              │
├─────────────────────────────────────────────────────────┤
│ Port 3:  Pi-hole Backup RPi3 (192.168.10.11)            │
│          Access, Untagged VLAN 10, PVID 10              │
├─────────────────────────────────────────────────────────┤
│ Port 4:  NAS01 (192.168.10.20)                          │
│          Access, Untagged VLAN 10, PVID 10              │
├─────────────────────────────────────────────────────────┤
│ Port 5:  VM01 (192.168.10.21)                           │
│          Access, Untagged VLAN 10, PVID 10              │
├─────────────────────────────────────────────────────────┤
│ Port 6:  UniFi U6-Pro #1 (PoE+ powered)                 │
│          Trunk, Native VLAN 11 (U), Tagged: 20,21 (T)   │
│          SSIDs: Secure, Guest, IoT                      │
├─────────────────────────────────────────────────────────┤
│ Port 7:  UniFi U6-Pro #2 (PoE+ powered)                 │
│          Trunk, Native VLAN 11 (U), Tagged: 20,21 (T)   │
│          SSIDs: Secure, Guest, IoT                      │
├─────────────────────────────────────────────────────────┤
│ Port 8:  Management                                     │
│          Access, Untagged VLAN 1, PVID 1                │
└─────────────────────────────────────────────────────────┘
```

## Verification Steps

After configuration, verify the following:

### 1. Check VLAN Membership Matrix

Navigate to: **VLAN → 802.1Q → Advanced → VLAN Membership**

**Expected Configuration (T=Tagged, U=Untagged, -=Not Member):**

| Port | VLAN 1 | VLAN 10 | VLAN 11 | VLAN 20 | VLAN 21 |
|------|--------|---------|---------|---------|---------|
| 1 (Router em1) | U | T | T | T | T |
| 2 (Pi-hole Primary) | - | U | - | - | - |
| 3 (Pi-hole Backup) | - | U | - | - | - |
| 4 (NAS01) | - | U | - | - | - |
| 5 (VM01) | - | U | - | - | - |
| 6 (U6-Pro #1) | - | - | U | T | T |
| 7 (U6-Pro #2) | - | - | U | T | T |
| 8 (Management) | U | - | - | - | - |

Verify each VLAN shows:
- Correct member ports as shown above
- Tagged ports show "T"
- Untagged ports show "U"
- Excluded ports show "-"

### 2. Check Port PVIDs

Navigate to: **VLAN → 802.1Q → Advanced → Port PVID Configuration**

**Expected PVIDs:**
- Port 1 (Router): PVID = 1
- Port 2 (Pi-hole Primary): PVID = 10
- Port 3 (Pi-hole Backup): PVID = 10
- Port 4 (NAS01): PVID = 10
- Port 5 (VM01): PVID = 10
- Port 6 (U6-Pro #1): PVID = 11
- Port 7 (U6-Pro #2): PVID = 11
- Port 8 (Management): PVID = 1

### 3. Check Port Status

Navigate to: **System → Management → Port Status** or **Switching → Ports → Port Status**

Verify:
- Connected ports show "Up" or "Link Up"
- Link speeds are correct (1000 Mbps for gigabit)
- Duplex mode is "Full"

### 4. Test Connectivity

From OPNsense router via console or SSH:
```bash
# Test interfaces are up
ifconfig | grep em

# Should see:
# em0 (WAN)
# em1 (LAN/MGMT - native VLAN 1)
# em1.10 or em1_vlan10 (SERVERS - VLAN 10)
# em1.11 or em1_vlan11 (WIFI_SECURE - VLAN 11)
# em1.20 or em1_vlan20 (GUEST - VLAN 20)
# em1.21 or em1_vlan21 (HomeAssist - VLAN 21)

# Ping each network gateway from OPNsense
ping -c 1 192.168.1.1    # MGMT
ping -c 1 192.168.10.1   # SERVERS
ping -c 1 192.168.11.1   # WIFI_SECURE
ping -c 1 192.168.20.1   # GUEST
ping -c 1 192.168.21.1   # HomeAssist
```

### 5. Verify PoE Status

Navigate to: **System → PoE → PoE Port Status**

**Expected:**
- Port 6: Status = Delivering Power, Class = 4 (PoE+), Power ≈ 13-21W
- Port 7: Status = Delivering Power, Class = 4 (PoE+), Power ≈ 13-21W
- Total Power Budget: ~42W of 55W used

## Troubleshooting

See [TROUBLESHOOTING.md](TROUBLESHOOTING.md#switch-netgear-gs310tp) for switch issues.

---

## Best Practices

1. **Document Everything**: Label physical ports and keep this documentation updated
2. **Test Incrementally**: Configure one VLAN at a time and test before moving to next
3. **Backup Configuration**: Export switch config after successful setup
4. **Management Access**: Always keep one port in VLAN 1 for management access
5. **Port Security**: Enable port security features after initial setup
6. **Firmware Updates**: Keep switch firmware up-to-date

## Advanced Configuration (Optional)

### Link Aggregation (LAG)
If you have multiple links between switch and router, configure LAG:
- Increases bandwidth
- Provides redundancy
- Requires configuration on both switch and OpenWRT

### QoS (Quality of Service)
Prioritize traffic types:
- VoIP → Highest priority
- Video streaming → High priority
- File transfers → Normal priority
- Guest traffic → Low priority

### IGMP Snooping
Enable for multicast traffic (streaming, IPTV):
- **Switching → Multicast → IGMP Snooping**
- Enable globally and per-VLAN

### Port Mirroring
For network monitoring with IDS:
- Mirror trunk port to monitoring port
- Connect IDS/monitoring device

## Configuration Backup

**Export Configuration:**
1. Go to: **Maintenance → File Management → Save Configuration**
2. Download and store safely
3. Include date and version in filename

**Restore Configuration:**
1. Go to: **Maintenance → File Management → Restore Configuration**
2. Upload saved configuration file
3. Switch will reboot with restored settings

## Summary

After completing this configuration:
- ✅ Switch handles all 5 VLANs (1, 10, 11, 20, 21)
- ✅ Router trunk port (1) carries all VLANs to OPNsense em1
- ✅ Server ports (2-5) are access ports on VLAN 10
- ✅ AP trunk ports (6-7) carry wireless VLANs to UniFi U6-Pro APs
- ✅ Management port (8) is access port on VLAN 1
- ✅ PoE+ powers both UniFi APs (42W of 55W budget)
- ✅ Multiple SSIDs mapped to VLANs: Secure (11), Guest (20), IoT (21)
- ✅ Router enforces all security policies between networks

## Next Steps

1. **Setup UniFi Controller**: See [10_UNIFI_AP_SETUP.md](10_UNIFI_AP_SETUP.md)
2. **Adopt and configure APs**: Create SSIDs mapped to VLANs
3. **Test wireless clients**: Verify correct VLAN assignment per SSID
4. **Update floating rules**: Add HomeAssist to firewall rules
