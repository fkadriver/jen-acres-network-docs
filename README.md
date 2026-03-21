# Home Network Configuration - OPNsense

Complete network configuration for x86 quad-port router running **OPNsense 26.1** (FreeBSD-based) with VLAN segmentation, Pi-hole DNS filtering, Tailscale subnet routing, and comprehensive security policies.

> **Note**: Previous OpenWRT configurations are preserved in [`archive/openwrt/`](archive/openwrt/) for reference.

## Network Architecture

**Supernet**: `192.168.0.0/16` (all internal networks)

### Networks

| Network | Interface | Port | Purpose | Supernet | DHCP | Key IPs |
|---------|-----------|------|---------|----------|------|---------|
| 192.168.1.0/24 | MGMT | em1 (VLAN 1 trunk) | Management | Secure_Net | Disabled | Router: .1, Switch: .2 |
| 192.168.10.0/24 | SERVERS | em1 (VLAN 10 trunk) | Server Infrastructure | Secure_Net | Enabled | Router: .1, Pi-hole: .10, .11 |
| 192.168.11.0/24 | WIFI_SECURE | em1 (VLAN 11 trunk) | Wireless Secured | Secure_Net | Enabled | Router: .1 |
| 192.168.20.0/24 | GUEST | em1 (VLAN 20 trunk) | Guest Access | Unsecure_Net | Enabled | Router: .1 |
| 192.168.21.0/24 | HomeAssist | em1 (VLAN 21 trunk) | Home Automation/IoT | Unsecure_Net | Enabled | Router: .1 |
| 192.168.30.0/24 | Cailin | em1 (VLAN 30 trunk) | Personal (Cailin) | Unsecure_Net | Enabled | Router: .1 |
| 192.168.254.0/24 | DMZ | Switch port 23 | DMZ — direct modem, bypasses router | — | Modem (.254) | From modem |

**Supernet Aliases (for simplified firewall rules):**
- **Secure_Net**: 192.168.0.0/20 (MGMT, SERVERS, WIFI_SECURE)
- **Unsecure_Net**: 192.168.16.0/20 (GUEST, HomeAssist, Cailin)

**WAN**: 192.168.254.0/24 (DHCP from 192.168.254.254)

## Hardware

- **Router**: Protectli FW41-0-4-32 (192.168.1.1)
  - Fanless x86_64 mini PC
  - 4GB RAM, 32GB mSATA SSD
  - 4x Intel Gigabit Ethernet ports:
    - `em0`: WAN (to ISP modem/router)
    - `em1`: LAN/trunk (to Aruba switch — carries all VLANs)
    - `em2`: Unused
    - `em3`: Break-glass emergency access (192.168.99.0/29)
- **Switch**: HPE Aruba 2530-24G PoE+ J9773A (192.168.1.2, managed, VLAN-capable, 195W PoE+)
  - Port 1: Router trunk (VLANs 1, 10, 11, 20, 21, 30)
  - Port 2: NAS01 (VLAN 10)
  - Ports 3, 5: Pi-holes (VLAN 10)
  - Port 4: VM01 (VLAN 10)
  - Ports 13-14: UniFi APs (PoE+, trunk VLANs 10/11/20/21)
  - Port 23: DSL Modem (VLAN 250 DMZ — bypasses router)
  - Port 24: Management Laptop (VLAN 1)
  - Port 25 (SFP): NetGear GS310TP dumb switch (VLAN 11)
- **WiFi**: 2x Ubiquiti UniFi U6-Pro (802.11ax, PoE+ powered via switch)
  - U6Basement (192.168.10.61, port 13), U6MainLevel (192.168.10.60, port 14)
  - Managed via UniFi Controller on vm01 (192.168.10.21)
  - SSIDs: Secure (VLAN 11), Guest (VLAN 20), IoT (VLAN 21)
- **Pi-hole Primary**: Raspberry Pi 4 (192.168.10.10, DNS server)
- **Pi-hole Backup**: Raspberry Pi 3 (192.168.10.11, redundant DNS server)

## Features

- ✅ **Network Segmentation**: 5 isolated networks via VLANs
- ✅ **UniFi WiFi**: Centrally managed U6-Pro APs with multiple SSIDs
- ✅ **Tailscale Integration**: Subnet router advertising `192.168.0.0/16`, ACLs managed via git
- ✅ **Pi-hole DNS**: Network-wide ad blocking with redundant DNS (primary + backup)
- ✅ **Security Hardening**: Admin users, SSH keys, 2FA, disabled root login
- ✅ **Security Policies**:
  - MGMT network internet access via Tailscale exit nodes only
  - Guest network fully isolated
  - Home Automation/IoT isolated from secure networks
  - Wireless Secured with general internet access
  - Cross-network rules for specific services
- ✅ **HTTPS Certificates**: Free Let's Encrypt certificates via Tailscale
- ✅ **Centralized Logging**: All devices forward logs to syslog server via Tailscale
- ✅ **Monitoring Ready**: Netflow/IDS monitoring capability

## Quick Start

1. **Install OPNsense** → [docs/01_OPNSENSE_INSTALLATION.md](docs/01_OPNSENSE_INSTALLATION.md)
2. **Setup Tailscale** → [docs/02_TAILSCALE_SETUP.md](docs/02_TAILSCALE_SETUP.md)
3. **Configure VLANs** → [docs/03_VLAN_CONFIG.md](docs/03_VLAN_CONFIG.md)
4. **Configure Switch** → [docs/04_SWITCH_CONFIG.md](docs/04_SWITCH_CONFIG.md) — Aruba 2530-24G switch setup
5. **Firewall rules** → [docs/05_FIREWALL_RULES.md](docs/05_FIREWALL_RULES.md)
6. **Pi-hole setup** → [docs/06_PIHOLE_SETUP.md](docs/06_PIHOLE_SETUP.md)
7. **UniFi AP setup** → [docs/07_UNIFI_AP_SETUP.md](docs/07_UNIFI_AP_SETUP.md)
8. **Security hardening** → [docs/08_SECURITY_HARDENING.md](docs/08_SECURITY_HARDENING.md)

## Repository Structure

```
.
├── README.md                              # This file
├── docs/                                  # Documentation (numbered for order)
│   ├── 01_OPNSENSE_INSTALLATION.md        # OPNsense installation guide
│   ├── 02_TAILSCALE_SETUP.md              # Tailscale subnet router, tags, logging
│   ├── 03_VLAN_CONFIG.md                  # VLAN configuration
│   ├── 04_SWITCH_CONFIG.md                # HPE Aruba 2530-24G switch configuration
│   ├── 05_FIREWALL_RULES.md               # Firewall rules reference
│   ├── 06_PIHOLE_SETUP.md                 # Pi-hole setup (primary + backup, iOS app, syslog)
│   ├── 07_UNIFI_AP_SETUP.md               # UniFi U6-Pro access point setup
│   ├── 08_SECURITY_HARDENING.md           # Security hardening (users, SSH, 2FA)
│   └── TROUBLESHOOTING.md                 # Consolidated troubleshooting guide
├── tailnet/                               # Tailscale ACL policy (git submodule)
│   ├── policy.hujson                      # Tailscale ACL configuration
│   └── README.md                          # ACL documentation
├── archive/                               # Archived configurations
│   └── openwrt/                           # Previous OpenWRT setup (archived)
└── .gitignore                             # Git ignore rules
```

## Design Benefits

### Centralized WiFi with UniFi
- **Single Management**: All APs managed via UniFi Controller
- **Multiple SSIDs**: One AP serves Secure, Guest, and IoT networks via VLANs
- **PoE Powered**: No separate power adapters needed
- **Enterprise Features**: Band steering, airtime fairness, seamless roaming

### Supernetting (192.168.0.0/16)
- **Single Tailscale Route**: One advertised route covers all networks
- **Simplified Routing**: Cleaner routing tables
- **Easy Expansion**: Add new subnets within the supernet without route updates
- **Efficient**: Reduces routing overhead

### Supernet Security Architecture
- **Simplified Firewall Rules**: Two /20 supernets (Secure_Net, Unsecure_Net) enable simple security policies
- **Clear Security Boundaries**: Isolated networks in Unsecure_Net, trusted networks in Secure_Net
- **Single Rule Blocking**: Block Unsecure_Net → Secure_Net with one floating rule
- **Efficient Routing**: Tailscale advertises single /16 covering all networks
- **Easy Expansion**: Add new networks within existing supernets without firewall rule changes

### Management Network Security
- **Isolated Management**: MGMT network for router and switch management only
- **Tailscale-Only Internet**: MGMT network internet access via Tailscale exit nodes
- **Static IPs**: No DHCP on MGMT network for tighter control
- **Emergency Access**: Console access always available (keyboard/monitor)

## Security Notes

- **Admin Users**: Dedicated admin users with SSH keys and 2FA (root login disabled)
- **MGMT Network**: No direct internet access; use Tailscale exit nodes for updates
- **Network Segmentation**: Secure_Net (MGMT, SERVERS, WIFI_SECURE) vs Unsecure_Net (GUEST, HomeAssist)
- **Isolated Networks**: Guest and Home Automation networks cannot access secure networks
- **IoT Isolation**: Home Automation devices on separate VLAN from guest and secure networks
- **Pi-hole DNS**: All DNS queries filtered for ads/malware (192.168.10.10)
- **Firewall**: Default deny with floating rules for simplified policy management
- **HTTPS**: Valid Let's Encrypt certificates via Tailscale (no browser warnings)
- **Centralized Logging**: All devices log to `syslog-server` (DNS alias in Pi-hole)

## Backup and Recovery

### OPNsense Configuration Backup

**Via Web UI**: System → Configuration → Backups
- Download configuration regularly
- Store securely with encryption
- Test restore procedure periodically

### Emergency Access

If you lose web UI access:
1. **Console Access**: Connect keyboard/monitor to router
2. **Reset Admin Password**: Option 3 from console menu
3. **Reconfigure Network**: Option 2 from console menu
4. **Restore Backup**: Upload via web UI after regaining access

See individual OPNsense documentation for detailed troubleshooting.

## Contributing

This is a personal infrastructure configuration. Fork for your own use.

## License

MIT License - See LICENSE file
