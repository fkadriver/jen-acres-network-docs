# OPNsense Floating Rules Configuration

Floating rules in OPNsense are processed **before** interface-specific rules and can apply to multiple interfaces simultaneously. This is ideal for implementing network-wide security policies with minimal rule duplication.

## Why Floating Rules for This Network?

Your supernet architecture enables simplified firewall management:

**Network Segmentation:**
- **Secure_Net** (192.168.0.0/20): MGMT, SERVERS, WIFI_SECURE
- **Unsecure_Net** (192.168.16.0/20): GUEST, HomeAssist, Boys

**Benefits:**
- ✅ **8 floating rules** instead of 20+ interface rules
- ✅ **Centralized policy management**
- ✅ **Consistent security across all networks**
- ✅ **Easier to audit and maintain**

---

## Step 1: Create Firewall Aliases

Before creating floating rules, create aliases for easier management.

**Navigate:** Firewall → Aliases

### Alias 1: Secure_Net

**Click:** + Add

- **Name**: `Secure_Net`
- **Type**: Network(s)
- **Content**: `192.168.0.0/20`
- **Description**: `Secure networks (MGMT, SERVERS, WIFI_Secure)`

**Click:** Save

---

### Alias 2: Unsecure_Net

**Click:** + Add

- **Name**: `Unsecure_Net`
- **Type**: Network(s)
- **Content**: `192.168.16.0/20`
- **Description**: `Isolated networks (GUEST, HomeAssist, Boys)`

**Click:** Save

---

### Alias 3: Pi_hole_DNS

**Click:** + Add

- **Name**: `Pi_hole_DNS`
- **Type**: Host(s)
- **Content**:
  ```
  192.168.10.10
  192.168.10.11
  ```
- **Description**: `Pi-hole DNS servers (primary + backup)`

**Click:** Save

**Note:** This alias includes both the primary Pi-hole (RPi4 at 192.168.10.10) and backup Pi-hole (RPi3 at 192.168.10.11). If you're not using a backup Pi-hole, only include 192.168.10.10.

---

### Alias 4: Internal_Network

**Click:** + Add

- **Name**: `Internal_Network`
- **Type**: Network(s)
- **Content**: `192.168.0.0/19`
- **Description**: `All internal networks (covers both Secure_Net and Unsecure_Net)`

**Click:** Save

**Note:** This /19 network (192.168.0.0 - 192.168.31.255) encompasses:
- Secure_Net: 192.168.0.0/20 (192.168.0.0 - 192.168.15.255)
- Unsecure_Net: 192.168.16.0/20 (192.168.16.0 - 192.168.31.255)

This alias is useful for rules that need to match all internal traffic regardless of security level.

---

### Alias 5: WAN_Net

**Click:** + Add

- **Name**: `WAN_Net`
- **Type**: Network(s)
- **Content**: `192.168.254.0/24`
- **Description**: `ISP network (upstream modem/router)`

**Click:** Save

**Note:** This alias represents your ISP's network where the modem/router resides (192.168.254.254). Blocking client VLAN access to this network prevents:
- Unauthorized access to ISP modem management interface
- Reconnaissance of upstream network infrastructure
- Potential attacks against ISP equipment

Only the MGMT VLAN should have access to the ISP modem if needed for troubleshooting.

---

### Alias 6: UniFi_Controller

**Click:** + Add

- **Name**: `UniFi_Controller`
- **Type**: Host(s)
- **Content**: `192.168.10.21`
- **Description**: `UniFi Network Application (vm01 - may change)`

**Click:** Save

**Note:** This is the server running the UniFi Network Application (Docker on vm01). Update this alias if the controller moves to a different host.

---

### Alias 7: AP_Communication

**Click:** + Add

- **Name**: `AP_Communication`
- **Type**: Port(s)
- **Content**:
  ```
  8080
  3478
  10001
  8443
  ```
- **Description**: `UniFi AP communication ports`

**Click:** Save

**Port Reference:**
| Port | Protocol | Purpose |
|------|----------|---------|
| 8080 | TCP | Device inform/communication |
| 3478 | UDP | STUN (NAT traversal) |
| 10001 | UDP | Device discovery |
| 8443 | TCP | Controller web UI (HTTPS) |

---

**Click:** Apply Changes (after creating all aliases)

---

### Auto-Generated Interface Network Aliases

**Note:** OPNsense automatically creates network aliases when you configure interfaces. You do NOT need to manually create these:

- **MGMT net** → 192.168.1.0/24 (auto-generated from LAN/MGMT interface)
- **SERVERS net** → 192.168.10.0/24 (auto-generated from SERVERS interface)
- **WIFI_SECURE net** → 192.168.11.0/24 (auto-generated from WIFI_SECURE interface)
- **GUEST net** → 192.168.20.0/24 (auto-generated from GUEST interface)

These auto-generated aliases will be used in the floating rules below. The naming format is: `[Interface_Name] net`

---

## Step 2: Create Floating Rules

**Navigate:** Firewall → Rules → Floating

**IMPORTANT:** Floating rules are processed **top-to-bottom**. Order matters!

### Floating Rule 1: Allow web UI access from MGMT and Tailscale only (Highest Priority)

**Click:** + Add (top right)

**General Settings:**
- **Action**: Pass
- **Quick**: ✓ (Apply action immediately on match)
- **Interface**: Select all management interfaces:
  - ✓ MGMT_LAN
  - ✓ Tailscale
- **Direction**: in
- **TCP/IP Version**: IPv4

**Source:**
- **Source**: Managment_Access

**Destination:**
- **Destination**: This Firewall
- **Destination port range**: HTTPS

**Extra Options:**
- **Protocol**: TCP
- **Description**: `Allow web UI access from MGMT and Tailscale only`
- **Category**: SECURE, UNSECURE, Tailscale, MGMT

**Click:** Save

---
### Floating Rule 2: Block web UI access from unauthorized networks

**Click:** + Add (top right)

**General Settings:**
- **Action**: Block
- **Quick**: ✓ (Apply action immediately on match)
- **Interface**: Select all interfaces:
  - ✓ SERVERS
  - ✓ WIFI_SECURE
  - ✓ GUEST
- **Destination**: This Firewall
- **Destination port range**: HTTPS

**Source:**
- **Source**: any

**Extra Options:**
- **Protocol**: TCP
- **Description**: `Block web UI access from unauthorized networks`
- **Category**: UNSECURE, MGMT

**Click:** Save

---

### Floating Rule 3: Allow SSH access from MGMT and Tailscale only

**Click:** + Add

**General Settings:**
- **Action**: Pass
- **Quick**: ✓
- **Interface**: Select management interfaces:
  - ✓ MGMT_LAN
  - ✓ Tailscale
- **Direction**: in
- **TCP/IP Version**: IPv4

**Source:**
- **Source**: Management_Access

**Destination:**
- **Destination**: This Firewall
- **Destination port range**: SSH (22)

**Extra Options:**
- **Protocol**: TCP
- **Description**: `Allow SSH access from MGMT and Tailscale only`
- **Category**: MGMT

**Click:** Save

---

### Floating Rule 4: Block SSH access from unauthorized networks

**Click:** + Add

**General Settings:**
- **Action**: Block
- **Quick**: ✓
- **Interface**: Select all client interfaces:
  - ✓ SERVERS
  - ✓ WIFI_SECURE
  - ✓ GUEST
- **Direction**: in
- **TCP/IP Version**: IPv4

**Source:**
- **Source**: any

**Destination:**
- **Destination**: This Firewall
- **Destination port range**: SSH (22)

**Extra Options:**
- **Protocol**: TCP
- **Description**: `Block SSH access from unauthorized networks`
- **Category**: Security

**Click:** Save

---

### Floating Rule 5: Allow DNS to Pi-hole

**Click:** + Add (top right)

**General Settings:**
- **Action**: Pass
- **Quick**: ✓ (Apply action immediately on match)
- **Interface**: Select all client interfaces:
  - ✓ SERVERS
  - ✓ WIFI_SECURE
  - ✓ GUEST
- **Direction**: in
- **TCP/IP Version**: IPv4

**Source:**
- **Source**: any

**Destination:**
- **Destination**: Single host or alias → Select `Pi_hole_DNS` (or enter 192.168.10.10)
- **Destination port range**: DNS (53) to DNS (53)

**Extra Options:**
- **Protocol**: TCP/UDP
- **Description**: `Allow all networks DNS access to Pi-hole`
- **Category**: DNS (optional)

**Click:** Save

---

### Floating Rule 6: Block Unsecure → Secure Networks

**Click:** + Add

**General Settings:**
- **Action**: Block
- **Quick**: ✓
- **Interface**: Select isolated networks:
  - ✓ GUEST
- **Direction**: in
- **TCP/IP Version**: IPv4

**Source:**
- **Source**: Select `Unsecure_Net`

**Destination:**
- **Destination**: Select `Secure_Net`

**Extra Options:**
- **Protocol**: any
- **Description**: `Block isolated networks from accessing secure networks`
- **Category**: Security (optional)
- **Log**: ✓ (optional - useful for troubleshooting)

**Click:** Save

---

### Floating Rule 7: Block All Client Networks from MGMT

**Click:** + Add

**General Settings:**
- **Action**: Block
- **Quick**: ✓
- **Interface**: Select all client interfaces:
  - ✓ SERVERS
  - ✓ WIFI_SECURE
  - ✓ GUEST
- **Direction**: in
- **TCP/IP Version**: IPv4

**Source:**
- **Source**: any

**Destination:**
- **Destination**: Select `MGMT net` (192.168.1.0/24) - auto-generated alias

**Extra Options:**
- **Protocol**: any
- **Description**: `Block all client networks from management network`
- **Category**: Security (optional)
- **Log**: ✓ (optional - useful for troubleshooting)

**Click:** Save

---

### Floating Rule 8: Block All Client Networks from WAN Network

**Click:** + Add

**General Settings:**
- **Action**: Block
- **Quick**: ✓
- **Interface**: Select all client interfaces:
  - ✓ SERVERS
  - ✓ WIFI_SECURE
  - ✓ GUEST
- **Direction**: in
- **TCP/IP Version**: IPv4

**Source:**
- **Source**: any

**Destination:**
- **Destination**: Select `WAN_Net` (192.168.254.0/24)

**Extra Options:**
- **Protocol**: any
- **Description**: `Block all client networks from ISP network`
- **Category**: Security (optional)
- **Log**: ✓ (optional - useful for troubleshooting)

**Click:** Save

---

**Click:** Apply Changes (after creating all floating rules)

---

## Step 3: Verify Floating Rule Order

After creating all rules, verify the order in **Firewall → Rules → Floating**:

**Correct Order (top to bottom):**

| # | Action | Quick | Interface | Source | Destination | Description |
|---|--------|-------|-----------|--------|-------------|-------------|
| 1 | Pass | ✓ | MGMT_LAN, Tailscale | Management_Access | This Firewall:443 | Allow web UI from MGMT/Tailscale |
| 2 | Block | ✓ | SERVERS, WIFI_SECURE, GUEST, HomeAssist, Boys | any | This Firewall:443 | Block web UI from unauthorized |
| 3 | Pass | ✓ | MGMT_LAN, Tailscale | Management_Access | This Firewall:22 | Allow SSH from MGMT/Tailscale |
| 4 | Block | ✓ | SERVERS, WIFI_SECURE, GUEST, HomeAssist, Boys | any | This Firewall:22 | Block SSH from unauthorized |
| 5 | Pass | ✓ | SERVERS, WIFI_SECURE, GUEST, HomeAssist, Boys | any | Pi_hole_DNS:53 | Allow all networks DNS to Pi-hole |
| 6 | Block | ✓ | GUEST, HomeAssist, Boys | Unsecure_Net | Secure_Net | Block isolated → secure |
| 7 | Block | ✓ | SERVERS, WIFI_SECURE, GUEST, HomeAssist, Boys | any | MGMT net | Block clients → MGMT |
| 8 | Block | ✓ | SERVERS, WIFI_SECURE, GUEST, HomeAssist, Boys | any | WAN_Net | Block clients → ISP network |

**Why This Order?**
1. **Web UI access first**: Allow management access from authorized networks
2. **Web UI block second**: Block unauthorized access to firewall management
3. **SSH access third**: Allow console access from authorized networks
4. **SSH block fourth**: Block unauthorized SSH access
5. **DNS fifth**: Allow critical DNS service before other blocking rules
6. **Network segmentation**: Block Unsecure → Secure
7. **Management protection**: Block all clients from MGMT
8. **ISP network protection**: Block all clients from accessing upstream ISP infrastructure (192.168.254.0/24 — modem management, not internet)

**Reorder if needed:** Drag and drop rules to change order.

---

## Step 4: Configure Per-Interface Rules

With floating rules handling security policies, per-interface rules become **very simple**.

### MGMT (LAN) Interface Rules

**Navigate:** Firewall → Rules → MGMT (or LAN)

**Rule 1: Allow MGMT to All**
- **Action**: Pass
- **Interface**: MGMT
- **Protocol**: any
- **Source**: MGMT net
- **Destination**: any
- **Description**: `Allow management network full access`

**Click:** Save → Apply Changes

---

### SERVERS (OPT1) and WIFI_SECURE (OPT2) Interface Rules

Both Secure_Net interfaces use the same single rule — floating rules handle MGMT and ISP modem protection.

**Navigate:** Firewall → Rules → [interface]

**Rule 1: Allow to Any**
- **Action**: Pass
- **Interface**: SERVERS *(repeat for WIFI_SECURE)*
- **Protocol**: any
- **Source**: *[interface]* net
- **Destination**: any
- **Description**: `Allow internet and LAN access (MGMT blocked by floating rule)`

**Click:** Save → Apply Changes

---

### GUEST (OPT3), HomeAssist (OPT4), Boys (OPT5) Interface Rules

All three Unsecure_Net interfaces use the same single rule — floating rules handle all isolation.

**Navigate:** Firewall → Rules → [interface]

**Rule 1: Allow to Any**
- **Action**: Pass
- **Interface**: GUEST *(repeat for HomeAssist, Boys)*
- **Protocol**: any
- **Source**: *[interface]* net
- **Destination**: any
- **Description**: `Allow internet access (isolation enforced by floating rules)`

**Click:** Save → Apply Changes

**Note:** Floating rules automatically:
- Block web UI access (Rule 2)
- Block SSH access (Rule 4)
- Allow DNS to Pi-hole (Rule 5)
- Block access to Secure_Net (Rule 6)
- Block access to MGMT (Rule 7)
- Block access to ISP modem (Rule 8)

So these networks get **internet only** + **DNS to Pi-hole**.

---

## Step 5: Verify NAT/Outbound Rules

**Navigate:** Firewall → NAT → Outbound

**Mode:** Ensure it's set to **Hybrid outbound NAT rule generation** (default)

**Verify automatic rules exist for all networks:**
- MGMT (LAN) → WAN
- SERVERS → WAN
- WIFI_SECURE → WAN
- GUEST → WAN
- HomeAssist → WAN
- Boys → WAN

These should be auto-generated. If missing, click **Save** to regenerate.

---

## Traffic Flow Examples

### Example 1: Guest Device Accessing Internet

**Device:** 192.168.20.50 (GUEST VLAN)
**Destination:** 8.8.8.8 (Google DNS)

**Rule Processing:**
1. **Floating Rule 1** (Web UI allow): No match (not MGMT/Tailscale interface)
2. **Floating Rule 2** (Web UI block): No match (destination is not firewall:443)
3. **Floating Rule 3** (DNS to Pi-hole): No match (destination is not Pi-hole)
4. **Floating Rule 4** (Block Unsecure → Secure): No match (8.8.8.8 is not in Secure_Net)
5. **Floating Rule 5** (Block → MGMT): No match (8.8.8.8 is not in MGMT net)
6. **Floating Rule 6** (Block → WAN_Net): No match (8.8.8.8 is not in WAN_Net)
7. **GUEST Interface Rule 1**: **MATCH** - Allow to any → **PASS**
8. **NAT Rule**: Traffic NATed to WAN IP
9. **Result**: ✅ Access allowed

---

### Example 2: Guest Device Accessing Pi-hole DNS

**Device:** 192.168.20.50 (GUEST VLAN)
**Destination:** 192.168.10.10:53 (Pi-hole DNS)

**Rule Processing:**
1. **Floating Rule 1** (Web UI allow): No match (not MGMT/Tailscale interface)
2. **Floating Rule 2** (Web UI block): No match (destination is not firewall:443)
3. **Floating Rule 3** (DNS to Pi-hole): **MATCH** (destination Pi-hole:53, Quick=✓) → **PASS**
4. Rules stop processing (Quick flag)
5. **Result**: ✅ DNS access allowed

**Note:** Even though Pi-hole (192.168.10.10) is in Secure_Net, the specific DNS allow rule (Rule 3) has higher priority due to ordering.

---

### Example 3: Guest Device Accessing Server

**Device:** 192.168.20.50 (GUEST VLAN)
**Destination:** 192.168.10.20 (Server in SERVERS VLAN)

**Rule Processing:**
1. **Floating Rule 1** (Web UI allow): No match (not MGMT/Tailscale interface)
2. **Floating Rule 2** (Web UI block): No match (destination is not firewall:443)
3. **Floating Rule 3** (DNS to Pi-hole): No match (destination is not Pi-hole DNS)
4. **Floating Rule 4** (Block Unsecure → Secure): **MATCH** (192.168.20.50 is in Unsecure_Net, 192.168.10.20 is in Secure_Net, Quick=✓) → **BLOCK**
5. Rules stop processing (Quick flag)
6. **Result**: ❌ Access denied

---

### Example 4: WiFi Device Accessing Server

**Device:** 192.168.11.50 (WIFI_SECURE VLAN)
**Destination:** 192.168.10.20 (Server in SERVERS VLAN)

**Rule Processing:**
1. **Floating Rule 1** (Web UI allow): No match (not MGMT/Tailscale interface)
2. **Floating Rule 2** (Web UI block): No match (destination is not firewall:443)
3. **Floating Rule 3** (DNS to Pi-hole): No match
4. **Floating Rule 4** (Block Unsecure → Secure): No match (WIFI_SECURE is not in Unsecure_Net)
5. **Floating Rule 5** (Block → MGMT): No match (192.168.10.20 is not in MGMT net)
6. **Floating Rule 6** (Block → WAN_Net): No match (192.168.10.20 is not in WAN_Net)
7. **WIFI_SECURE Interface Rule 1**: **MATCH** - Allow to any → **PASS**
8. **Result**: ✅ Access allowed

**Note:** WiFi can access servers because it's in Secure_Net, not blocked by floating rules.

---

### Example 5: Server Accessing Management Network

**Device:** 192.168.10.20 (SERVERS VLAN)
**Destination:** 192.168.1.2 (Switch in MGMT VLAN)

**Rule Processing:**
1. **Floating Rule 1** (Web UI allow): No match (not MGMT/Tailscale interface)
2. **Floating Rule 2** (Web UI block): No match (destination is not firewall:443)
3. **Floating Rule 3** (DNS to Pi-hole): No match
4. **Floating Rule 4** (Block Unsecure → Secure): No match (SERVERS is not in Unsecure_Net)
5. **Floating Rule 5** (Block → MGMT): **MATCH** (192.168.1.2 is in MGMT net, Quick=✓) → **BLOCK**
6. Rules stop processing (Quick flag)
7. **Result**: ❌ Access denied

**Note:** Even though SERVERS is in Secure_Net, the MGMT protection rule blocks all client VLANs.

---

### Example 6: WiFi Device Accessing ISP Modem

**Device:** 192.168.11.50 (WIFI_SECURE VLAN)
**Destination:** 192.168.254.254 (ISP Modem/Router)

**Rule Processing:**
1. **Floating Rule 1** (Web UI allow): No match (not MGMT/Tailscale interface)
2. **Floating Rule 2** (Web UI block): No match (destination is not firewall:443)
3. **Floating Rule 3** (DNS to Pi-hole): No match
4. **Floating Rule 4** (Block Unsecure → Secure): No match (WIFI_SECURE is not in Unsecure_Net)
5. **Floating Rule 5** (Block → MGMT): No match (192.168.254.254 is not in MGMT net)
6. **Floating Rule 6** (Block → WAN_Net): **MATCH** (192.168.254.254 is in WAN_Net, Quick=✓) → **BLOCK**
7. Rules stop processing (Quick flag)
8. **Result**: ❌ Access denied

**Note:** Client VLANs cannot access the ISP modem/router. Only MGMT VLAN has access for troubleshooting.

---

## Security Policy Summary

| Source Network | Can Access Internet | Can Access Pi-hole | Can Access SERVERS | Can Access WIFI_SECURE | Can Access MGMT | Can Access WAN_Net | Can Access GUEST |
|----------------|--------------------|--------------------|-------------------|----------------------|----------------|-------------------|----------------|
| **MGMT** | ✅ (via Tailscale) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **SERVERS** | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| **WIFI_SECURE** | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| **GUEST** | ✅ | ✅ (DNS only) | ❌ | ❌ | ❌ | ❌ | ✅ |

**Key Points:**
- ✅ **MGMT**: Full access to everything (management network + ISP modem)
- ✅ **Secure_Net** (SERVERS, WIFI_SECURE): Can access each other, internet, but not MGMT or WAN_Net
- ❌ **Unsecure_Net** (GUEST): Internet + DNS only, fully isolated from Secure_Net, MGMT, WAN_Net

---

## Troubleshooting

See [TROUBLESHOOTING.md](TROUBLESHOOTING.md#firewall-rules) for firewall rule issues.

---

## Maintenance Tips

### Adding New Networks

**To add a new network in Secure_Net:**
1. Ensure subnet is within 192.168.0.0/20 (e.g., 192.168.12.0/24)
2. Create interface in OPNsense (dedicated port or VLAN)
3. Add simple "allow to any" interface rule
4. Add to Floating Rules 2, 3, 5, 6 interface lists (Web UI block, DNS, MGMT block, WAN block)
5. Floating rules automatically handle security!

**To add a new network in Unsecure_Net:**
1. Ensure subnet is within 192.168.16.0/20 (e.g., 192.168.22.0/24)
2. Create interface in OPNsense (dedicated port or VLAN)
3. Add to Floating Rule 2 interface list (Web UI block)
4. Add to Floating Rule 3 interface list (for DNS access)
5. Add to Floating Rule 4 interface list (for isolation)
6. Add to Floating Rule 5 interface list (for MGMT protection)
7. Add to Floating Rule 6 interface list (for WAN_Net protection)
8. Add simple "allow to any" interface rule

---

### Modifying Security Policies

**To allow GUEST to access specific server (e.g., media server):**

Create a **new floating rule** ABOVE Rule 4:

- **Action**: Pass
- **Quick**: ✓
- **Interface**: GUEST
- **Source**: GUEST net
- **Destination**: Single host → specific server IP (e.g., 192.168.10.20)
- **Destination port**: Specific port (e.g., 8096 for Jellyfin)
- **Description**: `Allow GUEST to media server`

**Rule order becomes:**
1. Allow web UI from MGMT/Tailscale
2. Block web UI from unauthorized networks
3. Allow DNS to Pi-hole
4. **Allow GUEST → specific server (NEW)**
5. Block Unsecure → Secure
6. Block → MGMT
7. Block → WAN_Net

This allows the specific access while maintaining general isolation.

---

## Backup Configuration

After configuring floating rules:

**Navigate:** System → Configuration → Backups

**Click:** Download configuration

**Save as:** `opnsense-floating-rules-YYYYMMDD.xml`

Store securely with your other network documentation.

---

## Summary

This floating rules configuration provides:

✅ **Simplified Management**: 8 floating rules + 4 simple interface rules
✅ **Network Segmentation**: Secure_Net vs Unsecure_Net architecture
✅ **Centralized DNS**: All networks use Pi-hole
✅ **Web UI Protection**: Firewall management restricted to MGMT and Tailscale only
✅ **Management Protection**: MGMT network isolated from all clients
✅ **ISP Network Protection**: Client networks blocked from accessing ISP modem/router
✅ **Guest Isolation**: Guests can't access any internal resources (except DNS)
✅ **Flexible Security**: Easy to add exceptions for specific services

**Total Rules Required:**
- 8 floating rules (network-wide policies)
- 4 interface rules (1 per interface, all identical "allow to any")
- **= 12 total rules** instead of 20+ individual rules

This is the power of the supernet architecture combined with floating rules!
