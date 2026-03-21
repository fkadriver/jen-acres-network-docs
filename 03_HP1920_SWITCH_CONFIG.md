# HP 1920-8G-PoE+ Switch Configuration

Switch model: **HP 1920-8G-PoE+ (JG922A)**
Replaces: NetGear GS310TP (see [03_SWITCH_CONFIG.md](03_SWITCH_CONFIG.md) for legacy reference)

## Hardware Summary

| Specification | Value |
|---------------|-------|
| Model | HP 1920-8G-PoE+ (JG922A) |
| Ports | 8x GE RJ45 (all PoE+) + 2x SFP uplink |
| PoE Standard | 802.3at (PoE+) |
| PoE Budget | 180W total |
| Management | Web UI + SSH/Telnet (Comware 7 Lite) |
| Config upload | Maintenance → Config File → Restore |

### PoE Budget

| Device | Port | Max Draw |
|--------|------|----------|
| UniFi U6-Pro #1 | Port 6 | 21W |
| UniFi U6-Pro #2 | Port 7 | 21W |
| **Total used** | | **42W of 180W** |
| **Remaining** | | **138W** |

## Port Assignments

| Port | Device | Mode | Native VLAN | Tagged VLANs |
|------|--------|------|-------------|--------------|
| Gi1/0/1 | OPNsense em1 (router) | Trunk | 1 (MGMT) | 10, 11, 20, 21 |
| Gi1/0/2 | Pi-hole Primary | Access | 10 | — |
| Gi1/0/3 | Pi-hole Backup | Access | 10 | — |
| Gi1/0/4 | Server 3 | Access | 10 | — |
| Gi1/0/5 | VM01 | Access | 10 | — |
| Gi1/0/6 | UniFi U6-Pro #1 (PoE+) | Trunk | 11 | 20, 21 |
| Gi1/0/7 | UniFi U6-Pro #2 (PoE+) | Trunk | 11 | 20, 21 |
| Gi1/0/8 | Management | Access | 1 | — |
| Gi1/0/9 | SFP-1 (available) | — | — | Disabled |
| Gi1/0/10 | SFP-2 → Netgear GS310TP (dumb switch) | Access | 11 | — |

## VLAN Membership Matrix

| Port | VLAN 1 | VLAN 10 | VLAN 11 | VLAN 20 | VLAN 21 |
|------|--------|---------|---------|---------|---------|
| 1 (Router) | **U** | T | T | T | T |
| 2 (Pi-hole 1) | — | **U** | — | — | — |
| 3 (Pi-hole 2) | — | **U** | — | — | — |
| 4 (Server 3) | — | **U** | — | — | — |
| 5 (VM01) | — | **U** | — | — | — |
| 6 (U6-Pro #1) | — | — | **U** | T | T |
| 7 (U6-Pro #2) | — | — | **U** | T | T |
| 8 (Management) | **U** | — | — | — | — |
| 10 (Netgear dumb) | — | — | **U** | — | — |

U = Untagged (native/access), T = Tagged (trunk), — = Not a member

---

## Quick Start: Upload Pre-built Configuration

A ready-to-upload configuration file is provided at [`configs/hp1920-startup.cfg`](../configs/hp1920-startup.cfg). This is the fastest way to configure the switch.

### Steps

1. Connect a laptop to **Port 5** (or any port) and set a static IP `192.168.0.x/24`
2. Open browser to the switch's default IP (check label — typically `192.168.0.1` or `192.168.1.1`)
3. Log in with default credentials (admin / blank password — check label)
4. Navigate to: **Maintenance → Config File → Restore**
5. Upload `configs/hp1920-startup.cfg`
6. Switch reboots and applies the full configuration
7. Access the switch at `http://192.168.1.2` (new management IP)
8. **Set a strong admin password immediately**

> **Note:** After upload, the switch management IP changes to `192.168.1.2`. Reconnect your laptop to the same port or connect via the OPNsense network.

---

## Manual Configuration (Alternative)

If you prefer to configure the switch step-by-step via the web UI, follow these sections.

### 1. Initial Access and IP

**Default IP:** Check label — typically `192.168.0.1` or `192.168.1.1`

1. Connect laptop directly to any port
2. Set laptop to static IP in the switch's default subnet
3. Open browser to the switch's default IP
4. Log in, then **immediately set a strong admin password**

**Set management IP:**

**Navigation:** Network → IP → IPv4 Address

- IPv4 Address: `192.168.1.2`
- Subnet Mask: `255.255.255.0`
- Default Gateway: `192.168.1.1`

Click **Apply**. Reconnect to `http://192.168.1.2`.

### 2. Create VLANs

**Navigation:** Network → VLAN → Static VLAN

| VLAN ID | Name |
|---------|------|
| 1 | MGMT *(exists by default)* |
| 10 | Servers |
| 11 | WiFi_Secure |
| 20 | Guest |
| 21 | HomeAuto |

### 3. Configure Port 1 — Router Trunk

**Navigation:** Network → VLAN → Port

Select Port 1:
- Link type: **Trunk**
- PVID: **1**
- Add to VLANs: 1 (Untagged), 10, 11, 20, 21 (Tagged)

### 4. Configure Ports 2–5 — Server Access

For each port (2, 3, 4, 5):
- Link type: **Access**
- VLAN: **10**

### 5. Configure Port 8 — Management Access

- Link type: **Access**
- VLAN: **1**

### 6. Configure Ports 6–7 — AP Trunk (PoE+)

For each port (6, 7):
- Link type: **Trunk**
- PVID: **11**
- Add to VLANs: 11 (Untagged), 20, 21 (Tagged)
- **Do not** add VLANs 1 or 10

**Verify PoE:**

**Navigation:** PoE → PoE Port Configuration

- Ports 6 and 7: Admin Mode = **Enabled**, Power Mode = **802.3at**

---

## Verification

### VLAN Check

**Navigation:** Network → VLAN → Static VLAN

Confirm VLANs 1, 10, 11, 20, 21 exist and port memberships match the matrix above.

### PoE Status

**Navigation:** PoE → PoE Port Status

| Port | Expected Status | Expected Draw |
|------|----------------|---------------|
| 6 | Delivering Power | 13–21W |
| 7 | Delivering Power | 13–21W |

### Connectivity Test

From OPNsense shell (**Interfaces → Diagnostics → Ping**):

```bash
ping -c 1 192.168.1.2    # Switch management
ping -c 1 192.168.1.1    # MGMT gateway
ping -c 1 192.168.10.1   # SERVERS gateway
ping -c 1 192.168.11.1   # WIFI_SECURE gateway
ping -c 1 192.168.20.1   # GUEST gateway
ping -c 1 192.168.21.1   # HOMEAUTO gateway
```

---

## Configuration Backup

After setup, download the config and save it to this repo:

**Navigation:** Maintenance → Config File → Backup

Save as: `configs/hp1920-startup.cfg` (overwrite with the live version)

---

## Differences from NetGear GS310TP

| Feature | NetGear GS310TP | HP 1920-8G-PoE+ |
|---------|----------------|-----------------|
| PoE Budget | 55W | 180W |
| PoE Ports | 8 | 8 |
| SFP Uplinks | 0 | 2 |
| CLI Access | Web only | Web + SSH/Telnet |
| Config Upload | Yes (binary) | Yes (text — Comware) |
| Management | Netgear UI | Comware 7 Lite |

The VLAN design is identical — only the UI navigation paths change.

## Netgear GS310TP as Dumb Switch (Port 10 / SFP-2)

The old Netgear GS310TP is connected via SFP to Port 10, which is an access port on VLAN 11 (WiFi_Secure). The Netgear acts as a plain unmanaged-style switch — no VLANs configured on it, all ports carry VLAN 11 traffic transparently.

**Netgear configuration for dumb switch mode:**
- All ports: PVID 1, Untagged VLAN 1 (factory default — no changes needed)
- The HP switch handles all VLAN enforcement; the Netgear just extends port count for VLAN 11 devices

Devices connected to the Netgear will land on VLAN 11 (192.168.11.0/24) and receive IPs from OPNsense DHCP.

## Related Docs

- [`configs/hp1920-startup.cfg`](../configs/hp1920-startup.cfg) — Ready-to-upload config file
- [02_VLAN_CONFIG.md](02_VLAN_CONFIG.md) — VLAN and subnet reference
- [10_UNIFI_AP_SETUP.md](10_UNIFI_AP_SETUP.md) — UniFi AP adoption guide
- [06_FLOATING_RULES.md](06_FLOATING_RULES.md) — OPNsense firewall rules
