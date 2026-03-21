# Network Troubleshooting Guide

Consolidated troubleshooting for all network components.

---

## OPNsense

### Cannot Access Web Interface

```bash
# Check LAN IP from console
# Option 2: Set interface IP address
# Verify it's 192.168.1.1/24

# Check if web service is running
# Option 11: Restart web GUI
```

### Wrong Interface Assignments

```bash
# Re-run interface assignment
# Option 1: Assign Interfaces
# Swap em0 and em1 if needed
```

### Lost Root Password

```bash
# From console menu
# Option 3: Reset root password
```

---

## VLANs and Interfaces

### VLAN Interface Shows "Down"

```bash
# Check physical connection to switch
# Verify switch has VLANs configured on trunk port (Port 1)
# Check OPNsense: Interfaces → [VLAN] → Enable checkbox
```

### Cannot Ping Gateway from VLAN

```bash
# Verify DHCP is providing correct gateway
# Check firewall rules allow traffic
# Verify switch port is in correct VLAN
```

### Devices Get IP but No Internet

```bash
# Check NAT rules: Firewall → NAT → Outbound
# Verify WAN interface has internet connectivity
# Test from OPNsense: Interfaces → Diagnostics → Ping
```

### DNS Not Resolving

```bash
# Verify Pi-hole is accessible at 192.168.10.10
# Check firewall allows DNS traffic (port 53)
# Verify DHCP is providing correct DNS server
```

---

## Switch (NetGear GS310TP)

### Devices Can't Get IP Addresses

**Cause**: Port not in correct VLAN or PVID incorrect

**Solution**:
1. Verify port is member of the VLAN
2. Check PVID matches the VLAN ID
3. Ensure port is set to "Untagged" for that VLAN

### Trunk Port Not Passing All VLANs

**Cause**: VLANs not tagged on trunk port

**Solution**:
1. Go to VLAN Membership
2. Select trunk port
3. Add all VLANs as Tagged
4. Verify "T" appears for each VLAN

### UniFi AP Not Getting IP or Not Adopting

**Cause**: Trunk port misconfigured or PoE not delivering power

**Solution**:
1. Verify PoE is enabled and delivering power (System → PoE → PoE Port Status)
2. Check AP port (6 or 7) has correct VLAN membership (VLAN 11 untagged, 20/21 tagged)
3. Verify PVID is set to 11 for AP ports
4. Check DHCP is enabled on WIFI_SECURE interface in OPNsense
5. Verify firewall allows AP to reach UniFi Controller (port 8080)
6. Try factory reset of AP (hold reset 10 seconds) and re-adopt

### Switch Inaccessible After VLAN Changes

**Cause**: Management VLAN changed or port misconfigured

**Solution**:
1. Connect directly to switch using default VLAN 1
2. Reset switch to factory defaults (hold reset button 10 seconds)
3. Reconfigure switch from scratch

---

## Pi-hole

### Pi-hole Not Responding to DNS Queries

```bash
# Check if FTL is running
sudo systemctl status pihole-FTL

# Check listening ports
sudo netstat -tulpn | grep 53

# Check firewall (if enabled on Pi)
sudo iptables -L -n
```

### Clients Not Seeing Hostnames

1. Verify conditional forwarding is enabled (Settings → DNS → Reverse server)
2. Check OPNsense is providing hostnames via DHCP
3. Ensure reverse DNS zones are configured on OPNsense

### Tailscale Connection Issues

```bash
# Check Tailscale status
tailscale status

# Check if connected
tailscale ping pihole-primary

# View logs
sudo journalctl -u tailscaled -f
```

### "Ignoring Query from Non-Local Network"

If you see this error for 100.x.x.x addresses (Tailscale):
1. Go to **Settings → DNS**
2. Ensure **"Permit all origins"** is selected
3. Click **Save**

---

## Firewall Rules

### Guests Can't Access Internet

**Check:**
1. Floating Rule 3 order (DNS must be before blocking rules)
2. GUEST interface rule allows "any" destination
3. NAT rules exist for GUEST → WAN
4. Pi-hole is accessible (firewall log shows DNS allowed)

**Test:**
```bash
# From guest device
ping 8.8.8.8  # Should work
ping 192.168.10.1  # Should fail (Secure_Net blocked)
nslookup google.com  # Should work via Pi-hole
```

### WiFi Can't Access Servers

**Check:**
1. Both WIFI_SECURE and SERVERS are in Secure_Net (they should be)
2. Floating Rule 2 is NOT applied to WIFI_SECURE interface (should only be GUEST, HomeAssist)
3. WIFI_SECURE interface rule allows "any" destination

**Fix:**
- Edit Floating Rule 4 (Block Unsecure → Secure)
- Ensure Interface selection is ONLY: GUEST
- Do NOT include SERVERS or WIFI_SECURE

### DNS Not Working on Any VLAN

**Check:**
1. Pi-hole is running and accessible at 192.168.10.10
2. Floating Rule 3 (DNS to Pi-hole) is above the blocking rules
3. Floating Rule 3 has Quick=✓ enabled
4. DHCP is providing 192.168.10.10 as DNS server

**Test from OPNsense:**
```bash
# SSH to OPNsense
ping 192.168.10.10  # Pi-hole should respond
nslookup google.com 192.168.10.10  # Test DNS resolution
```

### Viewing Firewall Logs

**Navigate:** Firewall → Log Files → Live View

**Enable logging on floating rules** to see which rule is matching:
1. Edit each floating rule
2. Check "Log packets that are handled by this rule"
3. Save and apply

**Log interpretation:**
- **pass**: Traffic allowed
- **block**: Traffic blocked
- Shows which rule matched (by description)

---

## UniFi Access Points

### AP Not Adopting

1. **Check PoE**: Verify switch is delivering power
2. **Check VLAN**: Ensure trunk port configured correctly
3. **Check Firewall**: Allow AP to reach controller (port 8080)
4. **Check DNS**: Verify `unifi` hostname resolves (if using DNS discovery)
5. **Factory Reset**: Hold reset button 10 seconds, retry adoption

### Clients Not Getting VLAN IP

1. **Check SSID-VLAN mapping** in controller
2. **Check switch trunk** carries the VLAN
3. **Check OPNsense DHCP** is enabled for that VLAN
4. **Check firewall rules** allow DHCP traffic

### Poor WiFi Performance

1. **Channel Interference**: Use auto-channel or WiFi analyzer
2. **Transmit Power**: Reduce if APs are close together
3. **Band Steering**: Enable 5 GHz preference for capable devices
4. **Client Density**: Enable airtime fairness for many clients

### APs Stuck in "Adopting" State

1. Verify AP communication rule destination is opt1 (not opt3)
2. Check AP_Communication alias includes ports 3478, 8080, 10001
3. Verify UniFi controller is running: `ssh vm01 'docker ps | grep unifi'`
4. Check controller logs: `ssh vm01 'docker compose -f /home/scott/git/unifi_controller/docker-compose.yml logs -f'`

### Can't Access UniFi Web UI from WiFi_Secure

1. Verify port 8443 is in AP_Communication alias
2. Check floating rule allows traffic to opt1 on AP_Communication ports
3. Test direct access: `curl -k https://192.168.10.21:8443`

---

## HomeAssist / IoT

### HomeAssist Devices Not Getting IPs

1. Verify DHCP range exists for opt5 (192.168.21.100-250)
2. Check Dnsmasq is running: Services → Dnsmasq DNS & DHCP → General → Service status
3. Verify IoT SSID is mapped to VLAN 21 in UniFi controller

---

## General Diagnostics

### Check Interface Status

**Navigate:** Interfaces → Overview

Verify all interfaces show:
- Status: up
- IPv4: Correct IP address
- Media: active

### Test Connectivity from OPNsense

**Navigate:** Interfaces → Diagnostics → Ping

### Monitor Traffic in Real-Time

**Navigate:** Firewall → Log Files → Live View

### Check DHCP Leases

**Navigate:** Services → Dnsmasq DNS & DHCP → Leases

---

## Related Documentation

- [01_OPNSENSE_INSTALLATION.md](01_OPNSENSE_INSTALLATION.md) — Initial OPNsense setup
- [02_VLAN_CONFIG.md](02_VLAN_CONFIG.md) — VLAN and interface configuration
- [04_PIHOLE_SETUP.md](04_PIHOLE_SETUP.md) — Pi-hole DNS setup
- [06_FLOATING_RULES.md](06_FLOATING_RULES.md) — Firewall rules reference
- [10_UNIFI_AP_SETUP.md](10_UNIFI_AP_SETUP.md) — UniFi AP setup
