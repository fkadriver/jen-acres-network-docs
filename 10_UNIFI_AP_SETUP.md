# UniFi U6-Pro Access Point Setup

Configuration guide for Ubiquiti UniFi U6-Pro access points with VLAN-based multiple SSIDs.

## Overview

This upgrade replaces the dedicated router port WiFi architecture (Nighthawk MR70 on em2, Orbi on em3) with centrally-managed UniFi APs that support multiple VLANs via a single trunk connection.

**Benefits:**
- Multiple SSIDs on one AP (WIFI_SECURE, GUEST, HomeAssist)
- Centralized management via UniFi Controller
- Better coverage with enterprise-grade APs
- VLAN trunking eliminates need for dedicated router ports

## Hardware

| Device | Quantity | Location | Switch Port |
|--------|----------|----------|-------------|
| UniFi U6-Pro | 2 | TBD | Port 6, Port 7 |

### U6-Pro Specifications

- **WiFi**: WiFi 6 (802.11ax), 5.3 Gbps aggregate
- **Power**: 802.3at PoE+ required (13W typical, 21W max)
- **Ethernet**: 1x Gigabit RJ45
- **Coverage**: ~140 m² (1,500 ft²) per AP

## PoE Requirements

### NetGear GS310TP PoE+ Compatibility

The GS310TP is a PoE+ switch supporting 802.3at on all 8 ports:

| Specification | GS310TP | U6-Pro Requirement |
|---------------|---------|-------------------|
| PoE Standard | 802.3at (PoE+) | 802.3at (PoE+) |
| Per-Port Power | Up to 30W | 13-21W |
| Total PoE Budget | 55W | ~42W (2 APs max) |

**Compatibility: YES** - The GS310TP can power both U6-Pro APs directly.

### Power Budget Calculation

| Device | Port | Max Power |
|--------|------|-----------|
| U6-Pro #1 | Port 6 | 21W |
| U6-Pro #2 | Port 7 | 21W |
| **Total** | | **42W** |
| **GS310TP Budget** | | **55W** |
| **Remaining** | | **13W** |

**Note:** If you add PoE devices to other ports, monitor the total power budget in the switch management interface.

### Verify PoE Status

After connecting APs:

1. Go to: **System → PoE → PoE Port Configuration**
2. Verify ports 6 and 7 show:
   - Status: **Delivering Power**
   - Power Mode: **802.3at**
   - Power Drawn: ~13-21W each

## Network Architecture (Updated)

### VLAN Summary

| VLAN ID | Network | Subnet | Purpose | SSID |
|---------|---------|--------|---------|------|
| 11 | WIFI_SECURE | 192.168.11.0/24 | Trusted wireless | `YourNetwork` |
| 20 | GUEST | 192.168.20.0/24 | Guest access | `YourNetwork-Guest` |
| 21 | HomeAssist | 192.168.21.0/24 | Home automation/IoT | `YourNetwork-IoT` |

### Updated Topology

```
                    ┌───────────────────────────────────┐
                    │      Protectli FW41 Router        │
                    │          192.168.1.1              │
                    │                                   │
    WAN ───em0──────┤  WAN (DHCP from ISP)              │
                    │                                   │
    LAN ───em1──────┤  VLAN Trunk to Switch             │───── Trunk ───┐
                    │  Native: VLAN 1 (MGMT)            │                │
                    │  Tagged: 10, 11, 20, 21           │                │
                    │                                   │                │
    em2 ────────────┤  (Available - was WIFI_SECURE)    │                │
                    │                                   │                │
    em3 ────────────┤  (Available - was GUEST)          │                │
                    └───────────────────────────────────┘                │
                                                                         │
                    ┌────────────────────────────────────────────────────┘
                    │
              ┌─────▼────────────────────────────────────────────────┐
              │              NetGear GS310TP Switch                   │
              │                  192.168.1.2                         │
              ├──────────────────────────────────────────────────────┤
              │ Port 1:  Trunk to Router (VLANs 1,10,11,20,21)       │
              │ Port 2:  Pi-hole Primary (VLAN 10)                   │
              │ Port 3:  Pi-hole Backup (VLAN 10)                    │
              │ Port 4:  NAS01 (VLAN 10)                             │
              │ Port 5:  VM01 (VLAN 10)                              │
              │ Port 6:  UniFi U6-Pro #1 - Trunk (VLANs 11,20,21)    │
              │ Port 7:  UniFi U6-Pro #2 - Trunk (VLANs 11,20,21)    │
              │ Port 8:  Management (VLAN 1)                         │
              └──────────────────────────────────────────────────────┘
                           │                    │
                    ┌──────┴─────┐       ┌──────┴─────┐
                    │ U6-Pro #1  │       │ U6-Pro #2  │
                    │            │       │            │
                    │ SSIDs:     │       │ SSIDs:     │
                    │ - Secure   │       │ - Secure   │
                    │ - Guest    │       │ - Guest    │
                    │ - IoT      │       │ - IoT      │
                    └────────────┘       └────────────┘
```

## UniFi Controller Setup

### Deployment Options

| Option | Best For | Requirements |
|--------|----------|--------------|
| UniFi Network Application (self-hosted) | Full control, no cloud | Docker/VM on server |
| UniFi Cloud Key | Dedicated appliance | Hardware purchase |
| Hostifi (cloud-hosted) | No self-hosting | Monthly subscription |

### Option 1: Self-Hosted Docker (Recommended)

Deploy on a server in the SERVERS VLAN (192.168.10.x).

#### Docker Compose

Create `docker-compose.yml`:

```yaml
version: '3'
services:
  unifi-network-application:
    image: lscr.io/linuxserver/unifi-network-application:latest
    container_name: unifi-controller
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Chicago
      - MONGO_USER=unifi
      - MONGO_PASS=your-secure-password
      - MONGO_HOST=unifi-db
      - MONGO_PORT=27017
      - MONGO_DBNAME=unifi
    volumes:
      - ./unifi-config:/config
    ports:
      - 8443:8443  # Web UI (HTTPS)
      - 3478:3478/udp  # STUN
      - 10001:10001/udp  # Device discovery
      - 8080:8080  # Device communication
      - 1900:1900/udp  # L2 discovery (optional)
      - 8843:8843  # Guest portal HTTPS (optional)
      - 6789:6789  # Mobile throughput test (optional)
    restart: unless-stopped
    depends_on:
      - unifi-db

  unifi-db:
    image: mongo:4.4
    container_name: unifi-db
    volumes:
      - ./unifi-db:/data/db
    restart: unless-stopped
```

#### Start Controller

```bash
# Create directories
mkdir -p unifi-config unifi-db

# Start services
docker-compose up -d

# Check logs
docker-compose logs -f unifi-network-application
```

#### Access Controller

1. Open browser: `https://<server-ip>:8443`
2. Accept self-signed certificate warning
3. Complete initial setup wizard
4. Create admin account with strong password

### Initial Controller Configuration

#### Step 1: Site Settings

1. **Settings** → **Site**
2. Site Name: `Jen Acres` (or your preferred name)
3. Country: United States
4. Timezone: Your timezone

#### Step 2: Network Configuration

Create networks for each VLAN:

**Settings** → **Networks** → **Create New Network**

> **Note:** Because OPNsense is the router (not a UniFi gateway device), the IP address, subnet, and DHCP fields will be grayed out or unavailable. This is expected. UniFi only needs the VLAN ID to tag traffic — OPNsense handles all routing and DHCP.

##### WIFI_SECURE Network
- **Name**: WIFI_SECURE
- **Purpose**: Corporate
- **VLAN ID**: 11
- *(IP/DHCP fields are managed by OPNsense — leave as-is)*

##### GUEST Network
- **Name**: GUEST
- **Purpose**: Guest
- **VLAN ID**: 20
- *(IP/DHCP fields are managed by OPNsense — leave as-is)*
- **Guest Policies**: Enable client isolation (optional)

##### HomeAssist Network
- **Name**: HomeAssist
- **Purpose**: Corporate
- **VLAN ID**: 21
- *(IP/DHCP fields are managed by OPNsense — leave as-is)*

#### Step 3: WiFi Configuration

**Settings** → **WiFi** → **Create New WiFi Network**

##### Secure SSID
- **Name/SSID**: `YourNetwork`
- **Security**: WPA3/WPA2 (or WPA2 for older device compatibility)
- **Password**: Strong password
- **Network**: WIFI_SECURE (VLAN 11)
- **Band**: Both (2.4 GHz and 5 GHz)
- **Hide SSID**: No

##### Guest SSID
- **Name/SSID**: `YourNetwork-Guest`
- **Security**: WPA2
- **Password**: Simpler password for guests
- **Network**: GUEST (VLAN 20)
- **Band**: Both
- **Guest Policies**: Enable if desired

##### IoT SSID
- **Name/SSID**: `YourNetwork-IoT`
- **Security**: WPA2 (many IoT devices don't support WPA3)
- **Password**: Strong password
- **Network**: HomeAssist (VLAN 21)
- **Band**: 2.4 GHz only (better IoT compatibility)

## AP Adoption

### Method 1: Layer 2 Discovery (Same Subnet)

If the controller and APs are on the same subnet initially:

1. Connect APs to switch (ports 6-7)
2. APs appear in controller under **Devices**
3. Click **Adopt** for each AP
4. Wait for provisioning to complete

### Method 2: Layer 3 Adoption (Different Subnet)

Since APs will be on VLAN 11 and controller on VLAN 10:

#### Option A: SSH Adoption

1. Find AP IP address (check DHCP leases in OPNsense)
2. SSH to AP:
   ```bash
   ssh ubnt@<ap-ip>
   # Default password: ubnt
   ```
3. Set inform URL:
   ```bash
   set-inform http://192.168.10.x:8080/inform
   ```
   (Replace `192.168.10.x` with your controller IP)

#### Option B: DHCP Option 43

Configure OPNsense to provide UniFi controller IP via DHCP Option 43:

1. **Services** → **Dnsmasq DNS & DHCP** → **DHCP options**
2. Add option for WIFI_SECURE interface:
   - **Type**: Set
   - **Option**: 43
   - **Value**: `01:04:C0:A8:0A:0X` (hex-encoded controller IP)

   To encode IP `192.168.10.20`:
   - `01` = suboption 1 (UniFi controller)
   - `04` = length (4 bytes)
   - `C0:A8:0A:14` = 192.168.10.20 in hex

#### Option C: DNS Record

Create DNS record in Pi-hole:

1. **Local DNS** → **DNS Records**
2. Add: `unifi` → `192.168.10.x` (controller IP)

APs look for `unifi` hostname during discovery.

### Firewall Rules for Adoption

Ensure OPNsense allows AP-to-controller communication:

**Firewall** → **Rules** → **WIFI_SECURE** (or Floating Rules)

| Action | Source | Destination | Port | Description |
|--------|--------|-------------|------|-------------|
| Pass | WIFI_SECURE net | Controller IP | 8080/TCP | UniFi device inform |
| Pass | WIFI_SECURE net | Controller IP | 3478/UDP | STUN |
| Pass | WIFI_SECURE net | Controller IP | 10001/UDP | Device discovery |

## Post-Adoption Configuration

### AP Settings

For each adopted AP:

1. **Devices** → Click AP → **Settings**
2. **General**:
   - Name: `U6-Pro-Location1`, `U6-Pro-Location2`
3. **Radios**:
   - 2.4 GHz: Channel width 20 MHz, auto channel
   - 5 GHz: Channel width 80 MHz, auto channel
   - Transmit Power: Auto or Medium (avoid interference)
4. **Network**: Ensure management VLAN is set

### LED Configuration

- **Settings** → **Site** → **Device LED**
- Options: On, Off, or Off when adopted

## Verification

### Check VLAN Tagging

From a client connected to each SSID:

```bash
# Check IP address
ip addr    # Linux
ipconfig   # Windows

# Expected results:
# Secure SSID:  192.168.11.x
# Guest SSID:   192.168.20.x
# IoT SSID:     192.168.21.x
```

### Verify Isolation

From Guest network:
```bash
# Should work
ping 192.168.20.1    # Gateway
ping 8.8.8.8         # Internet

# Should NOT work (blocked by firewall)
ping 192.168.10.10   # Pi-hole direct
ping 192.168.11.x    # WIFI_SECURE
ping 192.168.1.1     # MGMT
```

### Controller Status

1. **Dashboard**: All APs show "Online"
2. **Devices**: No adoption errors
3. **Clients**: Devices appear on correct networks

## Troubleshooting

See [TROUBLESHOOTING.md](TROUBLESHOOTING.md#unifi-access-points) for UniFi AP issues.

## Maintenance

### Firmware Updates

1. **Settings** → **System** → **Firmware**
2. Enable auto-updates or manually update
3. Schedule updates during low-usage hours

### Controller Backup

1. **Settings** → **System** → **Backup**
2. Download backup regularly
3. Store securely

### Monitoring

- **Dashboard**: Real-time client count, throughput
- **Statistics**: Historical data
- **Alerts**: Configure email notifications

## Migration Notes

### Retiring Old Equipment

After UniFi APs are operational:

1. **Nighthawk MR70** (was on em2): Can be repurposed or retired
2. **Orbi** (was on em3): Can be repurposed or retired
3. **Router ports em2, em3**: Now available for other uses

### Clients Migration

1. Update saved WiFi credentials on all devices
2. Forget old SSIDs to force reconnection
3. IoT devices may need factory reset to connect to new SSID

## Summary

After completing this setup:
- 2x UniFi U6-Pro APs managed by UniFi Controller
- 3 SSIDs mapped to VLANs 11, 20, 21
- PoE powered via GS310TP switch
- Centralized management and monitoring
- Router ports em2/em3 freed for other uses
