# OPNsense Network Configuration Guide

## Network Design Overview

**Supernet Architecture:**
- **Secure_Net**: 192.168.0.0/20 (MGMT, SERVERS, WIFI_SECURE)
- **Unsecure_Net**: 192.168.16.0/20 (GUEST, HomeAssist)

**Key Design Feature**: All VLANs trunk through em1 on the Protectli to the Aruba switch. em3 is a dedicated, isolated break-glass port. em2 carries the Starlink secondary WAN (see [10_STARLINK_MULTIWAN.md](10_STARLINK_MULTIWAN.md)) — not a VLAN.
- **em1** → Aruba 2530-24G switch (single trunk, all VLANs)
- **em2** → Starlink secondary WAN (`opt9`)
- **em3** → Break-glass emergency access only (isolated /29, NOT in use for VLANs)

| Network | Interface | Physical | Purpose | Supernet | Gateway | DHCP Range |
|---------|-----------|----------|---------|----------|---------|------------|
| 192.168.1.0/24 | MGMT_LAN | em1 (native) | Management | Secure_Net | 192.168.1.1 | None (static) |
| 192.168.10.0/24 | SERVERS | em1 (VLAN 10) | Server Infrastructure | Secure_Net | 192.168.10.1 | 10.100-10.250 |
| 192.168.11.0/24 | WIFI_SECURE | em1 (VLAN 11) | Wireless Secured | Secure_Net | 192.168.11.1 | 11.100-11.250 |
| 192.168.20.0/24 | GUEST | em1 (VLAN 20) | Guest Access | Unsecure_Net | 192.168.20.1 | 20.100-20.250 |
| 192.168.21.0/24 | HomeAssist | em1 (VLAN 21) | HomeAssist/IoT | Unsecure_Net | 192.168.21.1 | 21.100-21.250 |
| 192.168.250.0/24 | DMZ | em1 (VLAN 250) | DMZ — router-managed, internet DNS, no internal access | — | 192.168.250.1 | 250.100-250.250 |
| 192.168.99.0/29 | MGMT_Only | em3 (isolated) | Break-glass emergency | — | 192.168.99.1 | 99.2-99.6 |

> Note: `192.168.254.0/24` is the **WAN-side** subnet Kinetic's modem hands to `em0`
> (see [10_STARLINK_MULTIWAN.md](10_STARLINK_MULTIWAN.md)) — it is unrelated to DMZ
> despite an older version of this doc set conflating the two.

> **VLAN 30 (Cailin, formerly "Boys") was decommissioned 2026-10-01** — removed from
> both the switch and OPNsense (interface, VLAN definition, DHCP range, firewall
> rules). Scott is rethinking this network's purpose entirely as part of the move to
> Starlink as primary WAN, rather than keeping it around unchanged. If it comes back,
> it'll likely look different from what's described in the sections below.

## Physical Topology

```
    T3200 Kinetic
    (DSL Modem)
         │
         │  (direct — bridge/passthrough mode; see research/T3200_bridge_mode.md)
         │
                    ┌───────────────────────────────────┐
                    │      Protectli FW41 Router        │
                    │          192.168.1.1              │
                    │                                   │
    WAN ───em0──────┤  WAN (IP from T3200 / ISP)        │
     (Intel I211)   │                                   │
                    │   em1: VLAN Trunk                 │
    Aruba──em1──────┤   Native: VLAN 1 (MGMT)          │
     (Intel I211)   │   Tagged: 10, 11, 20, 21, 30, 250 │
                    │                                   │
                    │   em2: Starlink secondary WAN     │
                    │       (opt9 — see 10_STARLINK_MULTIWAN.md) │
                    │                                   │
    [Break-Glass]   │   em3: 192.168.99.1/29            │
    Laptop──em3─────┤   (isolated — not for VLANs)     │
     (Intel I211)   │   DHCP: .2-.6 (5 addrs max)      │
                    └───────────────────────────────────┘
                         │
              ┌──────────┘
              │
    ┌─────────▼──────────────────────────────────────────┐
    │      HPE Aruba 2530-24G PoE+  (J9773A)             │
    │              192.168.1.2                            │
    ├─────────────────────────────────────────────────────┤
    │ Port 1:  Trunk → em1 (VLANs 1,10,11,20,21,30,250)  │
    │ Port 2:  NAS01         (VLAN 10)                    │
    │ Port 3:  Pi-hole Primary (VLAN 10)                  │
    │ Port 4:  VM01          (VLAN 10)                    │
    │ Port 5:  Pi-hole Backup  (VLAN 10)                  │
    │ Port 7:  NAS01 iDRAC   (VLAN 10)                    │
    │ Port 13: U6Basement PoE+ (VLAN 10+11+20+21)        │
    │ Port 14: U6MainLevel PoE+(VLAN 10+11+20+21)        │
    │ Port 23: Available     —                            │
    │ Port 24: Mgmt Laptop   (VLAN 1)                    │
    │ Port 25: NetGear (SFP) (VLAN 1 native)             │
    └─────────────────────────────────────────────────────┘
```

## Step-by-Step Network Configuration (From Scratch)

### Prerequisites
- OPNsense installed and accessible (see [01_OPNSENSE_INSTALLATION.md](01_OPNSENSE_INSTALLATION.md))
- Connected via **em1** (directly or through the Aruba switch) for initial configuration — em3 break-glass is configured in [01_OPNSENSE_INSTALLATION.md](01_OPNSENSE_INSTALLATION.md)
- Aruba 2530-24G switch configured with VLANs (see [04_SWITCH_CONFIG.md](04_SWITCH_CONFIG.md))

---

## Phase 1: Create VLANs

**Navigation**: Interfaces → Devices → VLAN

Create VLANs for all tagged networks. **Use `em1` as the parent interface**.

**IMPORTANT**: Do NOT create VLAN 1. em1 already carries VLAN 1 traffic natively (it is the LAN/MGMT interface).

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

### VLAN 250 - DMZ
- **Parent Interface**: em1
- **VLAN tag**: 250
- **Description**: DMZ
- Click **Save**

**Verify**: You should see 5 VLANs listed: `em1_vlan10`, `em1_vlan11`, `em1_vlan20`, `em1_vlan21`, `em1_vlan250`

---

## Phase 2: Assign All Interfaces

**Navigation**: Interfaces → Assignments

You should already see WAN (em0) and LAN (em1) assigned. LAN is already assigned to em1 from the initial console configuration. No reassignment needed. Add the VLAN interfaces below.

**Add SERVERS (VLAN 10):**
1. **Device**: Select `em1_vlan10` (or `vlan 0.10 on em1`)
2. Click **Add**

**Add WIFI_SECURE (VLAN 11):**
1. **Device**: Select `em1_vlan11`
2. Click **Add**

**Add GUEST (VLAN 20):**
1. **Device**: Select `em1_vlan20`
2. Click **Add**

**Add HomeAssist (VLAN 21):**
1. **Device**: Select `em1_vlan21`
2. Click **Add**

**Add DMZ (VLAN 250):**
1. **Device**: Select `em1_vlan250`
2. Click **Add**

Also add the Tailscale interface (`tailscale0`, once the `os-tailscale` plugin is
installed — see [02_TAILSCALE_SETUP.md](02_TAILSCALE_SETUP.md)) and em3 (break-glass,
already done in [01_OPNSENSE_INSTALLATION.md](01_OPNSENSE_INSTALLATION.md)).

> **OPT numbering isn't fixed**: OPNsense assigns `opt1`, `opt2`, etc. in whatever
> order you add interfaces — it is **not** guaranteed to match the table below on a
> fresh install. The table reflects this system's current live mapping (useful as a
> reference when using the web UI), not a sequence to reproduce exactly. What matters
> functionally is the VLAN tag and description, both of which this guide gets right.

| Interface | Identifier | Device |
|-----------|------------|--------|
| [WAN]     | wan        | em0 |
| [LAN]     | lan        | em1 — Native VLAN 1 (MGMT_LAN) |
| [WiFI_Secure] | opt1   | em1 VLAN 11 (WIFI_SECURE) |
| [GUEST]   | opt2       | em1 VLAN 20 (GUEST) |
| [SERVERS] | opt3       | em1 VLAN 10 (SERVERS) |
| [Tailscale] | opt4     | tailscale0 (Tailscale VPN) |
| [HomeAssist] | opt5    | em1 VLAN 21 (HomeAssist) |
| [MGMT_Only] | opt6     | em3 — Break-glass *(configured in 01_OPNSENSE_INSTALLATION.md)* |
| [DMZ]     | opt8       | em1 VLAN 250 (DMZ) |
| [STARLINK] | opt9      | em2 (Starlink secondary WAN — see [10_STARLINK_MULTIWAN.md](10_STARLINK_MULTIWAN.md)) |

Click **Save**

---

## Phase 3: Configure Each Interface

### LAN (MGMT - Native VLAN 1 on em1)

**Navigation**: Interfaces → LAN

- **Description**: MGMT_LAN
- **IPv4 address**: 192.168.1.1
- **Subnet mask**: 24

Click **Save** → **Apply Changes**

### SERVERS (VLAN 10)

**Navigation**: Interfaces → [the OPT slot you added SERVERS as — see table above]

- **Enable**: ✓
- **Description**: SERVERS
- **IPv4 Configuration Type**: Static IPv4
- **IPv4 address**: 192.168.10.1 / 24

Click **Save** → **Apply Changes**

### WIFI_SECURE (VLAN 11)

- **Enable**: ✓
- **Description**: WIFI_SECURE
- **IPv4 address**: 192.168.11.1 / 24

### GUEST (VLAN 20)

- **Enable**: ✓
- **Description**: GUEST
- **IPv4 address**: 192.168.20.1 / 24

### HomeAssist (VLAN 21)

- **Enable**: ✓
- **Description**: HomeAssist
- **IPv4 address**: 192.168.21.1 / 24

### DMZ (VLAN 250)

- **Enable**: ✓
- **Description**: DMZ
- **IPv4 address**: 192.168.250.1 / 24

> **DMZ firewall**: Block all traffic from 192.168.250.0/24 to Secure_Net (192.168.0.0/20) and Unsecure_Net (192.168.16.0/20). Allow WAN-bound traffic only. See [05_FIREWALL_RULES.md](05_FIREWALL_RULES.md).

---

## Phase 4: Configure DHCP Services (Dnsmasq)

OPNsense uses **Dnsmasq** for DHCP. DNS is handled by Pi-hole (Dnsmasq DNS port is disabled).

### Step 1: Enable Dnsmasq DHCP

**Navigation**: Services → Dnsmasq DNS & DHCP → General

Switch **Advanced options** dropdown to **"Advanced"**.

#### DNS Section
- **Listen port**: `0` (disables DNS — Pi-hole handles DNS)
- **DNSSEC**: ☐ Unchecked

#### DHCP Section
- **DHCP FQDN**: ✓ Checked
- **DHCP default domain**: `internal`
- **DHCP local domain**: ✓ Checked

Click **Save** → **Apply**

### Step 2: Configure DHCP Ranges

**Navigation**: Services → Dnsmasq DNS & DHCP → DHCP ranges

#### SERVERS (VLAN 10)
- **Interface**: SERVERS | **Start**: 192.168.10.100 | **End**: 192.168.10.250 | **Lease**: 86400

#### WIFI_SECURE (VLAN 11)
- **Interface**: WIFI_SECURE | **Start**: 192.168.11.100 | **End**: 192.168.11.250 | **Lease**: 86400

#### GUEST (VLAN 20)
- **Interface**: GUEST | **Start**: 192.168.20.100 | **End**: 192.168.20.250 | **Lease**: 7200

#### HomeAssist (VLAN 21)
- **Interface**: HomeAssist | **Start**: 192.168.21.100 | **End**: 192.168.21.250 | **Lease**: 86400

#### DMZ (VLAN 250)
- **Interface**: DMZ | **Start**: 192.168.250.100 | **End**: 192.168.250.250 | **Lease**: 3600

> **MGMT (VLAN 1)**: No DHCP — all management devices use static IPs.

#### MGMT_Only (em3 — Break-Glass)
- **Interface**: MGMT_Only | **Start**: 192.168.99.2 | **End**: 192.168.99.6 | **Lease**: 3600
- Subnet: 192.168.99.0/29 — only 5 addresses available by design
- A laptop plugged into em3 receives an IP automatically with no manual configuration needed

### Step 3: Configure DNS Servers via DHCP Options

**Navigation**: Services → Dnsmasq DNS & DHCP → DHCP options

Add two entries:

| Setting | Value |
|---------|-------|
| **Interface** | Any |
| **Option** | dns-server [6] |
| **Value** | 192.168.10.10,192.168.10.11 |
| **Description** | Pi-hole DNS servers (all VLANs) |

| Setting | Value |
|---------|-------|
| **Interface** | DMZ |
| **Option** | dns-server [6] |
| **Value** | 1.1.1.2,1.0.0.2 |
| **Description** | Cloudflare 1.1.1.1 for Families (malware blocking) for DMZ |

> The DMZ-scoped option overrides the global entry for VLAN 250 clients. DMZ devices use [Cloudflare 1.1.1.1 for Families — malware only](https://blog.cloudflare.com/introducing-1-1-1-1-for-families/) (1.1.1.2 / 1.0.0.2). This blocks known malware domains without Pi-hole involvement.

Click **Save** → **Apply**

### Step 4: Add Static DHCP Mappings

**Navigation**: Services → Dnsmasq DNS & DHCP → Hosts

Add a host entry for each device that needs a stable IP:

| Hostname | MAC | IP | Interface |
|----------|-----|----|-----------|
| aruba2530 | (switch MAC — check label) | 192.168.1.2 | MGMT (LAN) |
| pihole01 | d8:3a:dd:29:b9:06 | 192.168.10.10 | SERVERS |
| pihole02 | b8:27:eb:d4:91:e6 | 192.168.10.11 | SERVERS |
| nas01 | 78:45:c4:22:63:89 | 192.168.10.20 | SERVERS |
| vm01 | d4:81:d7:5f:5d:dd | 192.168.10.21 | SERVERS |
| U6MainLevel | 0c:ea:14:81:bc:31 | 192.168.10.60 | SERVERS |
| U6Basement | 0c:ea:14:81:5c:75 | 192.168.10.61 | SERVERS |
| LBP162 (printer) | a0:c9:a0:32:3a:39 | 192.168.11.50 | WIFI_SECURE |

---

## Phase 5: Configure Firewall Rules

> **em3 break-glass is not configured here.** It is set up in
> [01_OPNSENSE_INSTALLATION.md](01_OPNSENSE_INSTALLATION.md) immediately after first login,
> before VLANs or any other configuration. By the time you reach this doc, em3 is already done.

See [05_FIREWALL_RULES.md](05_FIREWALL_RULES.md) for complete firewall configuration.

---

## Phase 6: Configure NAT/Outbound

**Navigation**: Firewall → NAT → Outbound

**Mode**: Hybrid outbound NAT rule generation (default)

Verify automatic rules exist for all VLANs → WAN.

---

## Verification and Testing

### 1. Check Interface Status

**Navigation**: Interfaces → Overview — all interfaces should show: Status: up, correct IPv4

### 2. Test VLAN Trunking

Check that devices on each VLAN can reach their gateway:

```bash
ping 192.168.1.1    # MGMT
ping 192.168.10.1   # SERVERS
ping 192.168.11.1   # WIFI_SECURE
ping 192.168.20.1   # GUEST
ping 192.168.21.1   # HomeAssist
ping 192.168.250.1  # DMZ
```

### 3. Verify VLAN Interfaces

Check that all VLAN interfaces appear under Interfaces → Overview and show Status: up with the correct IPv4 addresses. You should see em1_vlan10, em1_vlan11, em1_vlan20, em1_vlan21, em1_vlan250 in addition to LAN (em1) and WAN (em0).

### 4. Monitor Traffic

**Navigation**: Firewall → Log Files → Live View

---

## Backup Configuration

After completing setup:

**Navigation**: System → Configuration → Backups → Download configuration

The pre-commit hook strips private keys from XML before commit. See [.githooks/README.md](../.githooks/README.md).

---

## Troubleshooting

See [TROUBLESHOOTING.md](TROUBLESHOOTING.md#vlans-and-interfaces) for VLAN and interface issues.

---

## Next Steps

1. **Configure Aruba Switch**: See [04_SWITCH_CONFIG.md](04_SWITCH_CONFIG.md)
2. **Firewall rules**: See [05_FIREWALL_RULES.md](05_FIREWALL_RULES.md)
3. **Setup Pi-hole**: See [06_PIHOLE_SETUP.md](06_PIHOLE_SETUP.md)
4. **Setup UniFi APs**: See [07_UNIFI_AP_SETUP.md](07_UNIFI_AP_SETUP.md)
