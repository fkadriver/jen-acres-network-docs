# UniFi U6-Pro Access Point Setup

Configuration guide for Ubiquiti UniFi U6-Pro access points with VLAN-based multiple SSIDs.

## Overview

Centrally-managed UniFi APs supporting multiple VLANs via a single trunk connection to the Aruba 2530-24G PoE+ switch.

**Benefits:**
- Multiple SSIDs on one AP (WIFI_SECURE, GUEST, HomeAssist)
- Centralized management via UniFi Controller
- Better coverage with enterprise-grade APs
- VLAN trunking carries all wireless networks over a single uplink

## Hardware

| Device | Name | Location | Switch Port | IP | MAC |
|--------|------|----------|-------------|-----|-----|
| UniFi U6-Pro | U6Basement | Basement | Port 13 | 192.168.10.61 | 0c:ea:14:81:5c:75 |
| UniFi U6-Pro | U6MainLevel | Main Level | Port 14 | 192.168.10.60 | 0c:ea:14:81:bc:31 |

### U6-Pro Specifications

- **WiFi**: WiFi 6 (802.11ax), 5.3 Gbps aggregate
- **Power**: 802.3at PoE+ required (13W typical, 21W max)
- **Ethernet**: 1x Gigabit RJ45
- **Coverage**: ~140 m² (1,500 ft²) per AP

## PoE Requirements

### HPE Aruba 2530-24G PoE+ Compatibility

The HPE Aruba 2530-24G PoE+ (J9773A) supports 802.3at on all 24 RJ45 ports with 195W total budget:

| Specification | HPE Aruba 2530-24G PoE+ | U6-Pro Requirement |
|---------------|------------------------|-------------------|
| PoE Standard | 802.3af/at (PoE+) | 802.3at (PoE+) |
| Per-Port Power | Up to 30W | 13-21W |
| Total PoE Budget | 195W | ~42W (2 APs) |

**Compatibility: YES** - The Aruba 2530-24G powers both U6-Pro APs on ports 13 and 14.

### Power Budget Calculation

| Device | Port | Max Power |
|--------|------|-----------|
| U6-Pro #1 | Port 13 | 21W |
| U6-Pro #2 | Port 14 | 21W |
| **Total** | | **42W** |
| **Aruba 2530-24G Budget** | | **195W** |
| **Remaining** | | **153W** |

**Note:** If you add PoE devices to other ports, monitor the total power budget under **PoE → PoE Status** in the switch web UI.

### Verify PoE Status

After connecting APs:

1. Go to: **PoE → PoE Status** (or **PoE → PoE Configuration**)
2. Verify ports 13 and 14 show:
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
    em2 ────────────┤  (Unused)                         │                │
                    │                                   │                │
    em3 ────────────┤  Break-glass (192.168.99.0/29)   │                │
                    └───────────────────────────────────┘                │
                                                                         │
                    ┌────────────────────────────────────────────────────┘
                    │
              ┌─────▼────────────────────────────────────────────────┐
              │       Aruba 2530-24G PoE+ (J9773A)                   │
              │                  192.168.1.2                         │
              ├──────────────────────────────────────────────────────┤
              │ Port 1:  Trunk to Router (VLANs 1,10,11,20,21)       │
              │ Port 2:  NAS01 (VLAN 10)                             │
              │ Port 3:  Pi-hole Primary (VLAN 10)                   │
              │ Port 4:  VM01 (VLAN 10)                              │
              │ Port 5:  Pi-hole Backup (VLAN 10)                    │
              │ Port 13: U6Basement  - Native VLAN 10, Tagged 11,20,21│
              │ Port 14: U6MainLevel - Native VLAN 10, Tagged 11,20,21│
              │ Port 24: Management Laptop (VLAN 1)                  │
              │ Port 25: GS310TP dumb switch SFP (VLAN 11)          │
              └──────────────────────────────────────────────────────┘
                           │                    │
                    ┌──────┴──────┐      ┌──────┴──────┐
                    │ U6Basement  │      │ U6MainLevel │
                    │ 10.61       │      │ 10.60       │
                    │ SSIDs:      │      │ SSIDs:      │
                    │ - Secure    │      │ - Secure    │
                    │ - Guest     │      │ - Guest     │
                    │ - IoT       │      │ - IoT       │
                    └─────────────┘      └─────────────┘
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

### Management Network: VLAN 10 (SERVERS)

APs are managed on VLAN 10 (native/untagged on switch ports 13-14), placing them on the same subnet as the UniFi controller (192.168.10.21). This enables reliable L2 discovery with no cross-VLAN firewall rules required.

DHCP Option 43 (`http://192.168.10.21:8080/inform`) is set globally in OPNsense dnsmasq, so APs auto-discover the controller on first boot.

### Adoption Steps

1. Connect APs to switch ports 13 and 14 (PoE powers them)
2. APs boot and get IPs on VLAN 10 (192.168.10.x) via DHCP
3. APs auto-inform the controller via DHCP Option 43
4. Open controller: `https://192.168.10.21:8443`
5. **Devices** → **Adopt** each AP
6. Wait for provisioning (~2 min per AP)

### Static DHCP Reservations (OPNsense dnsmasq)

Set fixed IPs so APs always get the same address:

| Name | IP | MAC |
|------|----|-----|
| U6Basement | 192.168.10.61 | 0c:ea:14:81:5c:75 |
| U6MainLevel | 192.168.10.60 | 0c:ea:14:81:bc:31 |

**OPNsense:** Services → Dnsmasq → Static Leases → add both entries.

### Firewall Notes

No additional firewall rules are needed — APs and controller are on the same VLAN 10 subnet. The floating rule "Allow AP to connect to UniFi" (WIFI_SECURE → controller) is disabled and can be removed.

## Post-Adoption Configuration

### AP Settings

For each adopted AP:

1. **Devices** → Click AP → **Settings**
2. **General**:
   - Name: `U6Basement`, `U6MainLevel`
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

## Summary

After completing this setup:
- 2x UniFi U6-Pro APs managed by UniFi Controller
- 3 SSIDs mapped to VLANs 11, 20, 21
- PoE powered via Aruba 2530-24G PoE+ switch (J9773A), ports 13 and 14
- Centralized management and monitoring
