# Switch Configuration — HPE Aruba 2530-24G PoE+ (J9773A)

## Hardware Summary

| Specification | Value |
|---|---|
| Model | HPE Aruba 2530-24G PoE+ |
| Part Number | J9773A |
| Ports | 24x GE RJ45 (all PoE+) + 2x SFP uplinks (ports 25-26) |
| PoE Standard | 802.3af/at (PoE+), 195W total |
| Management | Web UI + CLI (SSH/Telnet/Console) |
| Management IP | 192.168.1.2 (VLAN 1) |
| Default IP | 192.168.0.1 (check label) |
| Factory Reset | Hold Clear button 10+ seconds |

---

## Port Assignments

| Port | Device | Mode | VLAN |
|------|--------|------|------|
| 1 | Router (em1) | Trunk | Native VLAN 1; Tagged 10, 11, 20, 21 |
| 2 | NAS01 | Access | VLAN 10 |
| 3 | Pi-hole Primary | Access | VLAN 10 |
| 4 | VM01 | Access | VLAN 10 |
| 5 | Pi-hole Backup | Access | VLAN 10 |
| 6-12 | Available | — | — |
| 13 | U6Basement AP (PoE+) | Trunk | Native VLAN 10; Tagged 11, 20, 21 |
| 14 | U6MainLevel AP (PoE+) | Trunk | Native VLAN 10; Tagged 11, 20, 21 |
| 15-23 | Available | — | — |
| 24 | Management Laptop | Access | VLAN 1 |
| 25 | NetGear GS310TP (SFP uplink) | Access | VLAN 11 |
| 26 | Available (SFP) | — | — |

---

## VLAN Membership Matrix

| Port | VLAN 1 | VLAN 10 | VLAN 11 | VLAN 20 | VLAN 21 |
|------|--------|---------|---------|---------|---------|
| 1 (Router) | **U** | T | T | T | T |
| 2 (NAS01) | — | **U** | — | — | — |
| 3 (Pi-hole 1) | — | **U** | — | — | — |
| 4 (VM01) | — | **U** | — | — | — |
| 5 (Pi-hole 2) | — | **U** | — | — | — |
| 6-12 (Available) | **U** | — | — | — | — |
| 13 (U6Basement) | — | **U** | T | T | T |
| 14 (U6MainLevel) | — | **U** | T | T | T |
| 15-23 (Available) | **U** | — | — | — | — |
| 24 (Mgmt Laptop) | **U** | — | — | — | — |
| 25 (NetGear SFP) | — | — | **U** | — | — |
| 26 (SFP unused) | — | — | — | — | — |

U = Untagged (native), T = Tagged, — = Not a member

---

## PoE Budget

| Device | Port | Draw |
|--------|------|------|
| U6-Pro Basement | Port 13 | 21W |
| U6-Pro Main Level | Port 14 | 21W |
| **Total** | | **42W of 195W** |

---

## Initial Setup — Factory Reset

1. Hold **Clear** button on front panel 10+ seconds until LEDs flash — switch resets to factory defaults
2. Default management IP: 192.168.0.1 (static on VLAN 1)
3. Connect laptop to any port, set static IP 192.168.0.100/24
4. Access web UI: `http://192.168.0.1` — login: `admin` / (blank password)
5. Or connect via console cable and use CLI

---

## CLI Initial Setup (Bootstrap via console or SSH to 192.168.0.1)

```
# Set management IP
vlan 1
  ip address 192.168.1.2 255.255.255.0
  exit
ip default-gateway 192.168.1.1

# Set hostname
hostname aruba2530

# Set admin password
password manager
# Follow prompts to set password

# Save and verify
write memory
show ip
```

> After changing management IP to 192.168.1.2, reconnect to `http://192.168.1.2`.

---

## Web UI Configuration

Note that the Aruba 2530 web UI calls this "VLAN Management". Navigation paths use the Aruba ProVision web UI terminology.

### Step 1: Create VLANs

**Navigation**: VLAN → VLAN Management → Add VLAN

| VLAN ID | Name |
|---------|------|
| 1 | Default/MGMT *(exists by default)* |
| 10 | Servers |
| 11 | WiFi_Secure |
| 20 | Guest |
| 21 | HomeAuto |

### Step 2: Configure Port 1 — Router Trunk (all VLANs)

**Navigation**: VLAN → VLAN Management → select each VLAN → Port Configuration

> In Aruba ProVision, "Untagged" = native VLAN for that port (only one per port), "Tagged" = trunk VLAN.

For Port 1, set:

| VLAN | Port 1 Setting |
|------|---------------|
| 1 | Untagged |
| 10 | Tagged |
| 11 | Tagged |
| 20 | Tagged |
| 21 | Tagged |

### Step 3: Configure Ports 2, 3, 4, 5 — Server Access (VLAN 10)

For each port, remove from VLAN 1 and set VLAN 10 as Untagged:

| VLAN | Ports 2, 3, 4, 5 |
|------|-----------------|
| 1 | No (remove from VLAN 1) |
| 10 | Untagged |

### Step 4: Configure Ports 13, 14 — AP Trunk (PoE+)

For each AP port, VLAN 10 native + tagged 11, 20, 21:

| VLAN | Ports 13, 14 |
|------|-----------|
| 1 | No |
| 10 | Untagged |
| 11 | Tagged |
| 20 | Tagged |
| 21 | Tagged |

### Step 5: Configure Port 24 — Management Laptop (VLAN 1 access)

Port 24 stays on default VLAN 1 Untagged — no change needed (default).

### Step 6: Configure Port 25 (SFP) — NetGear Dumb Switch (VLAN 11)

| VLAN | Port 25 |
|------|---------|
| 1 | No |
| 11 | Untagged |

---

## CLI Equivalent (if preferred)

```
configure

# Create VLANs
vlan 10
  name "Servers"
vlan 11
  name "WiFi_Secure"
vlan 20
  name "Guest"
vlan 21
  name "HomeAuto"

# Port 1 — Router trunk
vlan 1 untagged 1
vlan 10 tagged 1
vlan 11 tagged 1
vlan 20 tagged 1
vlan 21 tagged 1

# Ports 2,3,4,5 — Server access (VLAN 10)
no vlan 1 untagged 2-5
vlan 10 untagged 2-5

# Ports 13,14 — AP trunk (native VLAN 10, tagged 11/20/21)
no vlan 1 untagged 13-14
vlan 10 untagged 13-14
vlan 11 tagged 13-14
vlan 20 tagged 13-14
vlan 21 tagged 13-14

# Port 25 (SFP) — NetGear dumb switch (VLAN 11 access)
no vlan 1 untagged 25
vlan 11 untagged 25

# Ports 6-12, 15-23 stay on VLAN 1 (default) — available

# Save
write memory
exit
```

---

## Enable PoE on Ports 13 and 14

**Navigation**: PoE → PoE Configuration

- Port 13: Admin: Enabled, Priority: High
- Port 14: Admin: Enabled, Priority: High

After connecting APs, verify power delivery under **PoE → PoE Status**.

---

## Verify VLAN Configuration

**Navigation**: VLAN → VLAN Management → select VLAN → Port Configuration

Or via CLI:

```
show vlan
show vlan 10
show vlans ports 1
show vlans ports 13
show poe status
```

Expected VLAN summary:

| ID | Untagged | Tagged |
|----|----------|--------|
| 1 | 1, 6-12, 15-24 | — |
| 10 | 2, 3, 4, 5, 13, 14 | 1 |
| 11 | 25 | 1, 13, 14 |
| 20 | — | 1, 13, 14 |
| 21 | — | 1, 13, 14 |

---

## Set Management IP (if not done via CLI)

**Navigation**: IP Configuration (under General or Management menu)

| Setting | Value |
|---------|-------|
| IP Address | 192.168.1.2 |
| Subnet Mask | 255.255.255.0 |
| Default Gateway | 192.168.1.1 |

---

## Save Configuration

**Navigation**: Management → Configuration → Save

Or CLI: `write memory`

---

## NetGear GS310TP as Dumb Switch (Port 25 SFP)

The NetGear is connected via Port 25 (SFP-1 uplink). It acts as a plain switch — no VLAN config needed on the NetGear itself. The Aruba enforces VLAN 11 access on Port 25. Devices connected to NetGear ports land on VLAN 11 (192.168.11.0/24, WiFi_Secure).

> Note: If using an SFP-to-RJ45 adapter, ensure compatibility with the Aruba 2530.

---

## Troubleshooting

- **Devices can't get IP**: Check port VLAN membership and that the correct VLAN is Untagged on that port
- **VLAN traffic not passing Port 1**: Verify VLAN is Tagged (not No) on Port 1; verify OPNsense em1 VLAN interfaces are up
- **PoE not delivering**: PoE → PoE Configuration → verify Admin = Enabled on ports 13/14; check 195W budget
- **Can't reach 192.168.1.2 after setup**: Ensure laptop is on MGMT VLAN (192.168.1.x); Port 24 should be VLAN 1 Untagged
- **Web UI unreachable**: Try CLI via console cable; verify `show ip` shows correct management IP
- **Configuration lost after reboot**: Run `write memory` after any configuration changes to persist settings

---

## Related Docs

- [03_VLAN_CONFIG.md](03_VLAN_CONFIG.md) — VLAN and subnet reference
- [07_UNIFI_AP_SETUP.md](07_UNIFI_AP_SETUP.md) — UniFi AP adoption
- [05_FIREWALL_RULES.md](05_FIREWALL_RULES.md) — OPNsense firewall rules
