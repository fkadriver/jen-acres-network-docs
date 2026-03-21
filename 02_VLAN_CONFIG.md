# OPNsense Network Configuration Guide

## Network Design Overview

**Supernet Architecture:**
- **Secure_Net**: 192.168.0.0/20 (MGMT, SERVERS, WIFI_SECURE)
- **Unsecure_Net**: 192.168.16.0/20 (GUEST, HomeAssist)

**Key Design Feature**: All VLANs trunk through em1 to the managed switch. UniFi U6-Pro APs on switch ports 6-7 receive VLAN-tagged traffic for multiple SSIDs (WIFI_SECURE, GUEST, HomeAssist). Router ports em2/em3 are available for future use.

| Network | Interface | Port | Purpose | Supernet | Gateway | DHCP Range |
|---------|-----------|------|---------|----------|---------|------------|
| 192.168.1.0/24 | MGMT | em1 (VLAN 1) | Management | Secure_Net | 192.168.1.1 | None (static) |
| 192.168.10.0/24 | SERVERS | em1 (VLAN 10) | Server Infrastructure | Secure_Net | 192.168.10.1 | 10.100-10.250 |
| 192.168.11.0/24 | WIFI_SECURE | em1 (VLAN 11) | Wireless Secured | Secure_Net | 192.168.11.1 | 11.100-11.250 |
| 192.168.20.0/24 | GUEST | em1 (VLAN 20) | Guest Access | Unsecure_Net | 192.168.20.1 | 20.100-20.250 |
| 192.168.21.0/24 | HomeAssist | em1 (VLAN 21) | HomeAssist/IoT | Unsecure_Net | 192.168.21.1 | 21.100-21.250 |

## Physical Topology

```
                    ┌───────────────────────────────────┐
                    │      Protectli FW41 Router        │
                    │          192.168.1.1              │
                    │                                   │
    WAN ───em0──────┤  192.168.254.x (DHCP from ISP)   │
     (Intel I211)   │                                   │
                    │   Gateway/NAT/Firewall            │
                    │                                   │
    LAN ───em1──────┤  VLAN Trunk (All Networks)       │───── Trunk ───┐
     (Intel I211)   │  Native: VLAN 1 (MGMT)            │                │
                    │  Tagged: 10, 11, 20, 21           │                │
                    │                                   │                │
   (avail)─em2─────┤  Available for future use         │                │
     (Intel I211)   │                                   │                │
                    │                                   │                │
   (avail)─em3─────┤  Available for future use         │                │
     (Intel I211)   │                                   │                │
                    └───────────────────────────────────┘                │
                                                                         │
                    ┌────────────────────────────────────────────────────┘
                    │
              ┌─────▼───────────────────────────────────────────────┐
              │              NetGear GS310TP Switch                  │
              │                  192.168.1.2                        │
              ├─────────────────────────────────────────────────────┤
              │ Port 1:  Trunk to Router (VLANs 1,10,11,20,21)      │
              │ Port 2:  Pi-hole Primary (VLAN 10)                  │
              │ Port 3:  Pi-hole Backup (VLAN 10)                   │
              │ Port 4:  NAS01 (VLAN 10)                            │
              │ Port 5:  VM01 (VLAN 10)                             │
              │ Port 6:  UniFi U6-Pro #1 - Trunk (VLANs 11,20,21)   │
              │ Port 7:  UniFi U6-Pro #2 - Trunk (VLANs 11,20,21)   │
              │ Port 8:  Management (VLAN 1)                        │
              └─────────────────────────────────────────────────────┘
                          │                     │
                   ┌──────┴──────┐       ┌──────┴──────┐
                   │ U6-Pro #1   │       │ U6-Pro #2   │
                   │             │       │             │
                   │ SSIDs:      │       │ SSIDs:      │
                   │ - Secure    │       │ - Secure    │
                   │ - Guest     │       │ - Guest     │
                   │ - IoT       │       │ - IoT       │
                   └─────────────┘       └─────────────┘
```

## Step-by-Step Network Configuration

### Prerequisites
- OPNsense installed and accessible at https://192.168.1.1
- Connected via em1 (LAN port) to configure
- Switch already configured with VLANs (see switch-config.md)

---

## Phase 1: Create VLANs

**Navigation**: Interfaces → Devices → VLAN

Create VLANs for all networks that use VLAN tagging over the trunk to the switch.

**IMPORTANT**: Do NOT create VLAN 1! The existing LAN interface (em1) already serves as the native/untagged VLAN 1 for management traffic.

### VLAN 10 - Servers
- **Parent Interface**: em1
- **VLAN tag**: 10
- **Description**: SERVERS
- Click **Save**

### VLAN 11 - WiFi Secure
- **Parent Interface**: em1
- **VLAN tag**: 11
- **Description**: WIFI_SECURE
- Click **Save**

### VLAN 20 - Guest
- **Parent Interface**: em1
- **VLAN tag**: 20
- **Description**: GUEST
- Click **Save**

### VLAN 21 - HomeAssist
- **Parent Interface**: em1
- **VLAN tag**: 21
- **Description**: HomeAssist
- Click **Save**

**Verify**: You should see 4 VLANs listed: `em1_vlan10`, `em1_vlan11`, `em1_vlan20`, `em1_vlan21`

---

## Phase 2: Assign All Interfaces

**Navigation**: Interfaces → Assignments

You should already see WAN (em0) and LAN (em1) assigned. Now add the remaining VLAN interfaces.

**Add SERVERS (VLAN 10):**
1. **Device**: Select the VLAN 10 interface (e.g., `vlan 0.10 on em1`)
2. Click **Add**

**Add WIFI_SECURE (VLAN 11):**
1. **Device**: Select the VLAN 11 interface (e.g., `vlan 0.11 on em1`)
2. Click **Add**

**Add GUEST (VLAN 20):**
1. **Device**: Select the VLAN 20 interface (e.g., `vlan 0.20 on em1`)
2. Click **Add**

**Add HomeAssist (VLAN 21):**
1. **Device**: Select the VLAN 21 interface (e.g., `vlan 0.21 on em1`)
2. Click **Add**

After adding all interfaces, you should see:

| Interface | Identifier | Device |
|-----------|------------|--------|
| [WAN]     | wan        | em0 |
| [LAN]     | lan        | em1 - Native VLAN 1 (MGMT) |
| [OPT1]    | opt1       | em1 VLAN 10 (SERVERS) |
| [OPT2]    | opt2       | em1 VLAN 11 (WIFI_SECURE) |
| [OPT3]    | opt3       | em1 VLAN 20 (GUEST) |
| [OPT4]    | opt4       | em1 VLAN 21 (HomeAssist) |

Click **Save** at the top of the page after adding all interfaces.

**Note**: Router ports em2 and em3 are available for future use.

---

## Phase 3: Configure Each Interface

### LAN (MGMT - Native VLAN 1 on em1)

**Navigation**: Interfaces → LAN

The LAN interface is already configured with 192.168.1.1/24. Optionally rename it for clarity:

- **Description**: Change from "LAN" to "MGMT" (optional but recommended)
- **IPv4 address**: 192.168.1.1 (already configured)
- **Subnet mask**: 24 (already configured)
- Leave all other settings as default

Click **Save** → **Apply Changes**

### OPT1 (SERVERS - VLAN 10 on em1)

**Navigation**: Interfaces → OPT1

- **Enable**: ✓ Enable interface
- **Description**: SERVERS
- **IPv4 Configuration Type**: Static IPv4
- **IPv6 Configuration Type**: None
- **MAC controls**: Leave empty
- **MTU**: Leave empty (default 1500)
- **MSS**: Leave empty

**Static IPv4 Configuration:**
- **IPv4 address**: 192.168.10.1
- **Subnet mask**: 24

Click **Save** → **Apply Changes**

### OPT2 (WIFI_SECURE - VLAN 11 on em1)

**Navigation**: Interfaces → OPT2

- **Enable**: ✓ Enable interface
- **Description**: WIFI_SECURE
- **IPv4 Configuration Type**: Static IPv4
- **IPv6 Configuration Type**: None
- **MAC controls**: Leave empty
- **MTU**: Leave empty (default 1500)
- **MSS**: Leave empty

**Static IPv4 Configuration:**
- **IPv4 address**: 192.168.11.1
- **Subnet mask**: 24

Click **Save** → **Apply Changes**

### OPT3 (GUEST - VLAN 20 on em1)

**Navigation**: Interfaces → OPT3

- **Enable**: ✓ Enable interface
- **Description**: GUEST
- **IPv4 Configuration Type**: Static IPv4
- **IPv6 Configuration Type**: None
- **MAC controls**: Leave empty
- **MTU**: Leave empty (default 1500)
- **MSS**: Leave empty

**Static IPv4 Configuration:**
- **IPv4 address**: 192.168.20.1
- **Subnet mask**: 24

Click **Save** → **Apply Changes**

### OPT4 (HomeAssist - VLAN 21 on em1)

**Navigation**: Interfaces → OPT4

- **Enable**: ✓ Enable interface
- **Description**: HomeAssist
- **IPv4 Configuration Type**: Static IPv4
- **IPv6 Configuration Type**: None
- **MAC controls**: Leave empty
- **MTU**: Leave empty (default 1500)
- **MSS**: Leave empty

**Static IPv4 Configuration:**
- **IPv4 address**: 192.168.21.1
- **Subnet mask**: 24

Click **Save** → **Apply Changes**

---

## Phase 4: Configure DHCP Services (Dnsmasq)

OPNsense 25.7+ includes **Dnsmasq** by default. Dnsmasq is lightweight, well-suited for networks under 1000 clients, and supports hostname registration.

### Step 1: Enable Dnsmasq DHCP

**Navigation**: Services → Dnsmasq DNS & DHCP → General

**IMPORTANT**: Change the **Advanced options** dropdown (top left) from "Basic" to **"Advanced"** to reveal all settings.

#### Basic Settings
- **Enable**: ✓ Checked
- **Interface**: Leave empty (DHCP ranges control which interfaces serve DHCP)
- **Interface [no DHCP]**: Nothing selected (this field is only visible in Advanced mode)

#### DNS Section
- **Listen port**: `0` (disables DNS - Pi-hole handles DNS)
- **DNSSEC**: ☐ Unchecked
- **No hosts lookup**: ☐ Unchecked

#### DNS Query Forwarding Section
- **Query DNS servers sequentially**: ✓ Checked
- **Require domain**: ☐ Unchecked
- **Do not forward to system defined DNS servers**: ☐ Unchecked
- **Do not forward private reverse lookups**: ☐ Unchecked

#### DHCP Section
- **DHCP FQDN**: ✓ Checked (registers hostnames)
- **DHCP default domain**: `internal` (or `lan`)
- **DHCP local domain**: ✓ Checked
- **DHCP authoritative**: ☐ Unchecked (unless you're the only DHCP server)
- **DHCP register firewall rules**: ✓ Checked

Click **Save** → **Apply**

### Step 2: Configure DHCP Ranges

**Navigation**: Services → Dnsmasq DNS & DHCP → DHCP ranges

Click **+** to add a new DHCP range for each network:

#### SERVERS (VLAN 10)

- **Interface**: SERVERS
- **Start address**: 192.168.10.100
- **End address**: 192.168.10.250
- **Lease time**: 86400 (24 hours in seconds)
- **Domain**: lan

Click **Save**

#### WIFI_SECURE (VLAN 11)

- **Interface**: WIFI_SECURE
- **Start address**: 192.168.11.100
- **End address**: 192.168.11.250
- **Lease time**: 86400
- **Domain**: lan

Click **Save**

#### GUEST (VLAN 20)

- **Interface**: GUEST
- **Start address**: 192.168.20.100
- **End address**: 192.168.20.250
- **Lease time**: 7200 (2 hours - shorter for guests)
- **Domain**: guest.lan

Click **Save**

#### HomeAssist (VLAN 21)

- **Interface**: HomeAssist
- **Start address**: 192.168.21.100
- **End address**: 192.168.21.250
- **Lease time**: 86400 (24 hours - IoT devices benefit from stable IPs)
- **Domain**: iot.lan

Click **Save**

### Step 3: Configure DNS Servers via DHCP Options

Dnsmasq DHCP uses **DHCP options** to set DNS servers (not in the range config).

**Navigation**: Services → Dnsmasq DNS & DHCP → DHCP options

Click **+** to add a DNS server option:

| Setting | Value |
|---------|-------|
| **Interface** | Any |
| **Type** | Set |
| **Option** | dns-server [6] |
| **Option6** | None |
| **Tag** | Nothing selected |
| **Value** | 192.168.10.10,192.168.10.11 |
| **Force** | ☐ Unchecked |
| **Description** | Pi-hole DNS servers |

Click **Save** → **Apply**

**Note**: Option `dns-server [6]` is DHCP option 6, which sets DNS servers for all DHCPv4 clients. The value is a comma-separated list of Pi-hole IPs. Option6 is for IPv6 (not used here).

#### Alternative: Per-interface DNS using Tags

To set different DNS per VLAN:

1. **Create a tag** in DHCP ranges → Tag [set] field (e.g., `servers`, `guest`)
2. **Create DHCP options** with the **Tag** field set to match

For most setups, a single dns-server option with Interface=Any works for all networks.

### MGMT Network

**Do NOT enable DHCP** for MGMT - Management devices should use static IPs:
- Router: 192.168.1.1
- Switch: 192.168.1.2
- Management PC: 192.168.1.x (static)

### Step 4: Apply Configuration

**Navigation**: Services → Dnsmasq DNS & DHCP → Settings

Click **Apply** to activate all DHCP settings.

### Hostname Registration with Pi-hole

Dnsmasq DHCP automatically tracks hostnames from DHCP requests. To make these hostnames resolvable via Pi-hole:

1. **Option A - Pi-hole Conditional Forwarding** (Recommended):
   - In Pi-hole: Settings → DNS → Conditional Forwarding
   - Enable conditional forwarding to OPNsense (192.168.10.1) for local domain "lan"
   - Pi-hole will query OPNsense for local hostnames

2. **Option B - Manual Local DNS Records**:
   - Add important hosts manually in Pi-hole: Local DNS → DNS Records
   - Best for critical infrastructure that needs guaranteed resolution

See [05_OPNSENSE_PIHOLE_INTEGRATION.md](05_OPNSENSE_PIHOLE_INTEGRATION.md) for detailed Pi-hole integration.

---

## Phase 5: Configure Firewall Rules

This network uses **floating rules** for simplified firewall management.

**See [06_FLOATING_RULES.md](06_FLOATING_RULES.md) for complete firewall configuration.**

---

## Phase 6: Configure NAT/Outbound

**Navigation**: Firewall → NAT → Outbound

**Mode**: Hybrid outbound NAT rule generation (default)

This will automatically create NAT rules for all VLAN interfaces to access the internet via WAN.

**Verify**: Check that automatic rules exist for:
- MGMT → WAN
- SERVERS → WAN
- WIFI_SECURE → WAN
- GUEST → WAN
- HomeAssist → WAN

---

## Verification and Testing

### 1. Check Interface Status

**Navigation**: Interfaces → Overview

Verify all interfaces show:
- Status: up
- IPv4: Correct IP address
- Media: active

### 2. Test Network Connectivity

From a device on each network:

```bash
# Test gateway connectivity
ping 192.168.X.1

# Test internet connectivity
ping 8.8.8.8

# Test DNS (after Pi-hole is configured)
nslookup google.com
```

### 3. Test Firewall Rules

**From WIFI_SECURE (192.168.11.x):**
```bash
# Should work
ping 192.168.11.1   # Gateway
ping 192.168.10.10  # Pi-hole
ping 192.168.10.20  # Servers (Secure_Net can access each other)
ping 8.8.8.8        # Internet

# Should NOT work
ping 192.168.1.1    # MGMT (blocked by floating rule)
ping 192.168.1.2    # Switch
```

**From GUEST (192.168.20.x):**
```bash
# Should work
ping 192.168.20.1   # Gateway
ping 8.8.8.8        # Internet
nslookup google.com # DNS via Pi-hole (port 53 allowed by floating rule)

# Should NOT work
ping 192.168.1.1    # MGMT (blocked by floating rule)
ping 192.168.10.10  # Pi-hole (except DNS port 53)
ping 192.168.10.20  # SERVERS (Unsecure_Net → Secure_Net blocked)
ping 192.168.11.1   # WIFI_SECURE (blocked)
```

**From HomeAssist (192.168.21.x):**
```bash
# Should work
ping 192.168.21.1   # Gateway
ping 8.8.8.8        # Internet
nslookup google.com # DNS via Pi-hole (port 53 allowed by floating rule)

# Should NOT work
ping 192.168.1.1    # MGMT (blocked by floating rule)
ping 192.168.10.10  # Pi-hole (except DNS port 53)
ping 192.168.10.20  # SERVERS (Unsecure_Net → Secure_Net blocked)
ping 192.168.11.1   # WIFI_SECURE (blocked)
ping 192.168.20.1   # GUEST (isolated - same supernet but separate network)
```

### 4. Monitor Traffic

**Navigation**: Firewall → Log Files → Live View

Watch traffic in real-time to verify rules are working correctly.

---

## Backup Configuration

After completing VLAN setup:

**Navigation**: System → Configuration → Backups

1. Click **Download configuration**
2. Save as: `opnsense-vlan-config-YYYYMMDD.xml`
3. Store securely in your backup location

---

## Troubleshooting

See [TROUBLESHOOTING.md](TROUBLESHOOTING.md#vlans-and-interfaces) for VLAN and interface issues.

---

## Next Steps

1. **Configure Switch for AP Trunk Ports**: See [03_SWITCH_CONFIG.md](03_SWITCH_CONFIG.md)
2. **Setup UniFi Access Points**: See [10_UNIFI_AP_SETUP.md](10_UNIFI_AP_SETUP.md)
3. **Configure Pi-hole**: See [05_OPNSENSE_PIHOLE_INTEGRATION.md](05_OPNSENSE_PIHOLE_INTEGRATION.md)
4. **Add static DHCP leases**: Services → Dnsmasq DNS & DHCP → Hosts (for static mappings)
5. **Configure port forwarding**: Firewall → NAT → Port Forward (if needed)
6. **Enable logging**: System → Settings → Logging
