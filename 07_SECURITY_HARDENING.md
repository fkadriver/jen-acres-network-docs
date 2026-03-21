# OPNsense Security Hardening Guide

## Overview

After initial installation, implement security best practices to harden your OPNsense router against unauthorized access.

## Critical Security Measures

✅ Create dedicated admin user (don't use root for daily tasks)
✅ Disable root SSH login
✅ Configure SSH key authentication
✅ Disable password authentication for SSH
✅ Change default web UI port (optional)
✅ Enable two-factor authentication (2FA)
✅ Configure automatic security updates

---

## Part 1: User Management

### Why Not Use Root?

**Security Risks of Using Root:**
- ❌ No audit trail (can't distinguish between admins)
- ❌ Single point of compromise
- ❌ Cannot implement role-based access control
- ❌ Common target for brute-force attacks
- ❌ Accidental damage (root has unlimited power)

**Best Practice:**
- ✅ Create individual admin accounts for each administrator
- ✅ Use root only for initial setup and emergencies
- ✅ Disable root login after admin user is created
- ✅ Enable detailed logging per user

---

## Part 2: Create Admin User

### Step 1: Create New Admin User

**Navigation**: System → Access → Users

**Click: + Add**

**User Settings:**
- **Username**: your-admin-username (e.g., `scott`, `admin1`)
- **Password**: Strong password (16+ characters, mixed case, numbers, symbols)
- **Full name**: Your Name
- **Login shell**: /bin/csh (default)
- **Expiration date**: Leave empty (never expires)

**Group Membership:**
- ✓ **admins** - Full administrative access

**Authorized keys** (optional for now, we'll set this up later):
- Leave empty for now

**OTP seed** (for 2FA):
- Leave empty (we'll configure 2FA later)

**Click: Save**

### Step 2: Test New Admin User

**IMPORTANT**: Test the new user BEFORE disabling root!

1. **Open new browser tab** (don't close existing session)
2. Go to: `https://192.168.1.1`
3. Login with:
   - Username: your-admin-username
   - Password: (the password you set)
4. Verify you can access all admin functions
5. Test SSH access (if enabled):
   ```bash
   ssh your-admin-username@192.168.1.1
   ```

**If login works**: Proceed to disable root
**If login fails**: Check user settings, group membership (must be in "admins")

---

## Part 3: Disable Root Login

### Disable Root Web UI Login

**Navigation**: System → Settings → Administration

**Authentication:**
- **Disable root login**: ✓ Check this box
  - Prevents root from logging into web interface
  - Root can still login via console (for emergencies)

**Click: Save**

### Disable Root SSH Login (Recommended)

**Edit SSH configuration:**

**Via Web UI**:
**Navigation**: System → Settings → Administration

**Secure Shell:**
- **Permit root user login**: ✗ Uncheck this box
  - Root cannot SSH in
  - Only admin users can SSH

**Click: Save**

---

## Part 4: SSH Key Authentication

### Why SSH Keys?

**Benefits:**
- ✅ Much more secure than passwords
- ✅ No password to brute-force
- ✅ Convenient (no typing password)
- ✅ Can be used with SSH agent

### Generate SSH Key Pair (On Your Workstation)

**On your local machine** (not OPNsense):

```bash
# Generate ED25519 key (modern, secure)
ssh-keygen -t ed25519 -C "opnsense-admin-$(whoami)"

# Save to: /home/scott/.ssh/opnsense_admin_ed25519
# Set a strong passphrase (protects key if stolen)
```

**View public key:**
```bash
cat ~/.ssh/opnsense_admin_ed25519.pub
```

**Copy the output** (starts with `ssh-ed25519 AAAA...`)

### Add Public Key to OPNsense User

**Navigation**: System → Access → Users

1. **Click on your admin user** (e.g., scott)
2. **Authorized keys** section:
   - Paste your public key (the entire line from `.pub` file)
3. **Click: Save**

### Test SSH Key Authentication

```bash
# Test SSH with key
ssh -i ~/.ssh/opnsense_admin_ed25519 your-admin-username@192.168.1.1

# Should login WITHOUT asking for password
# (May ask for key passphrase if you set one)
```

### Configure SSH Client for Convenience

**Edit `~/.ssh/config`:**

```bash
Host opnsense
    HostName 192.168.1.1
    User your-admin-username
    IdentityFile ~/.ssh/opnsense_admin_ed25519
    IdentitiesOnly yes

# Now you can simply:
# ssh opnsense
```

---

## Part 5: Disable Password Authentication (Optional - Advanced)

**CRITICAL**: Only do this AFTER confirming SSH key authentication works!

### Disable SSH Password Login

**Navigation**: System → Settings → Administration

**Secure Shell:**
- **Permit password login**: ✗ Uncheck this box
  - SSH will ONLY accept key authentication
  - No password-based SSH login allowed

**Click: Save**

**Test from external device:**
```bash
# Should work with key:
ssh -i ~/.ssh/opnsense_admin_ed25519 your-admin-username@192.168.1.1

# Should FAIL without key:
ssh randomuser@192.168.1.1
# Expected: Permission denied (publickey)
```

---

## Part 6: Two-Factor Authentication (2FA)

### Enable TOTP 2FA

**Navigation**: System → Access → Users

1. **Click on your admin user**
2. **OTP seed**:
   - Click **"Click to generate"**
   - QR code appears
3. **Scan QR code** with authenticator app:
   - Google Authenticator
   - Authy
   - 1Password
   - Bitwarden
4. **Click: Save**

### Test 2FA Login

1. **Logout** of OPNsense web UI
2. **Login** with:
   - Username: your-admin-username
   - Password: your-password
   - **Token**: (6-digit code from authenticator app)

**IMPORTANT**: Save backup codes in secure location!

### Configure 2FA for All Admin Users

Repeat for each admin user account.

---

## Part 7: Additional Security Measures

### Change Default Web UI Port (Optional)

**Navigation**: System → Settings → Administration

**Web GUI:**
- **TCP port**: Change from 443 to custom port (e.g., 8443)
  - Reduces automated scans
  - Adds security through obscurity (not primary security)

**Access URL becomes:**
```
https://192.168.1.1:8443
```

### Check for Firmware Updates Regularly

OPNsense does not have automatic firmware updates. Check manually on a regular schedule.

**Navigation**: System → Firmware → Updates

- Click **"Check for updates"** weekly
- Review changelog before updating
- Install security updates promptly
- Schedule maintenance windows for major upgrades

### Configure Firewall for Management Access

Restrict web UI and SSH access to authorized networks using floating rules.

#### Step 1: Create Management Access Alias

**Navigation**: Firewall → Aliases

**Click:** + Add

- **Name**: `Management_Access`
- **Type**: Network group
- **Content**: `MGMT__NET`
- **Description**: `Networks allowed to access firewall management`

**Click:** Save → Apply Changes

**Note:** `MGMT__NET` is an auto-generated alias for the MGMT interface network (192.168.1.0/24). When you set up Tailscale, you'll add the `Tailscale_Net` alias to this group. See [08_TAILSCALE_SETUP.md](08_TAILSCALE_SETUP.md).

#### Step 2: Create Floating Rules for Management Access

**Navigation**: Firewall → Rules → Floating

##### Rule 1: Allow Web UI from MGMT and Tailscale

**Click:** + Add

- **Action**: Pass
- **Quick**: ✓ Apply the action immediately on match
- **Interface**: Select all: MGMT, SERVERS, WIFI_SECURE, GUEST, Tailscale
- **Direction**: in
- **TCP/IP Version**: IPv4
- **Protocol**: TCP
- **Source**: `Management_Access` (the alias you created)
- **Destination**: This Firewall
- **Destination port range**: 443 (or your custom port like 8443)
- **Description**: Allow web UI access from MGMT and Tailscale only

**Click:** Save

##### Rule 2: Block Web UI from All Other Sources

**Click:** + Add (this rule must be BELOW Rule 1)

- **Action**: Block
- **Quick**: ✓ Apply the action immediately on match
- **Interface**: Select all: MGMT, SERVERS, WIFI_SECURE, GUEST, Tailscale
- **Direction**: in
- **TCP/IP Version**: IPv4
- **Protocol**: TCP
- **Source**: any
- **Destination**: This Firewall
- **Destination port range**: 443 (or your custom port)
- **Log**: ✓ (optional, to monitor blocked attempts)
- **Description**: Block web UI access from unauthorized networks

**Click:** Save

##### Rule 3: Allow SSH from MGMT and Tailscale (if SSH enabled)

**Click:** + Add

- **Action**: Pass
- **Quick**: ✓
- **Interface**: Select all interfaces
- **Direction**: in
- **TCP/IP Version**: IPv4
- **Protocol**: TCP
- **Source**: `Management_Access`
- **Destination**: This Firewall
- **Destination port range**: 22
- **Description**: Allow SSH access from MGMT and Tailscale only

**Click:** Save

##### Rule 4: Block SSH from All Other Sources

**Click:** + Add (below Rule 3)

- **Action**: Block
- **Quick**: ✓
- **Interface**: Select all interfaces
- **Direction**: in
- **TCP/IP Version**: IPv4
- **Protocol**: TCP
- **Source**: any
- **Destination**: This Firewall
- **Destination port range**: 22
- **Log**: ✓
- **Description**: Block SSH access from unauthorized networks

**Click:** Save → Apply Changes

#### Floating Rules Order

Ensure your floating rules are ordered correctly (drag to reorder if needed):

| # | Action | Source | Dest Port | Description |
|---|--------|--------|-----------|-------------|
| 1 | Pass | Management_Access | 443 | Allow web UI from MGMT/Tailscale |
| 2 | Block | any | 443 | Block web UI from others |
| 3 | Pass | Management_Access | 22 | Allow SSH from MGMT/Tailscale |
| 4 | Block | any | 22 | Block SSH from others |

#### Verification

Test from different networks:

```bash
# From MGMT network - should work
curl -k https://192.168.1.1:443

# From Tailscale - should work
curl -k https://192.168.1.1:443

# From SERVERS network - should be blocked
curl -k https://192.168.10.1:443

# From GUEST network - should be blocked
curl -k https://192.168.20.1:443
```

Check the firewall logs for blocked attempts:
**Navigation**: Firewall → Log Files → Live View

### Enable Logging and Monitoring

**Navigation**: System → Settings → Logging

**General Logging:**
- ✓ **Enable remote syslog**: Yes (see centralized logging below)
- **Log level**: Informational
- ✓ **Log firewall default blocks**: For security monitoring

**Navigation**: System → Settings → Administration

**Secure Shell:**
- ✓ **Enable SSH login logging**

### Configure Centralized Logging

Forward OPNsense logs to the centralized syslog server via Tailscale for security monitoring and analysis.

**Navigation**: System → Settings → Logging / targets

**Add new target:**

| Setting | Value |
|---------|-------|
| **Transport** | TCP(4) |
| **Applications** | (leave empty for all) |
| **Levels** | Warning and above |
| **Hostname** | `syslog-server` |
| **Port** | `514` |
| **Description** | Centralized Syslog Server |

**Click:** Save → Apply

**Note:** The `syslog-server` hostname is a DNS alias configured in Pi-hole, pointing to the actual syslog server on the Tailscale network. This provides a single point of configuration for all devices.

For complete centralized logging setup across all devices, see:
- [04_PIHOLE_SETUP.md - Local DNS Records](04_PIHOLE_SETUP.md#5-configure-local-dns-records) - Configure the `syslog-server` DNS alias
- [08_TAILSCALE_SETUP.md - Part 13](08_TAILSCALE_SETUP.md#part-13-centralized-logging-via-tailscale) - Full logging guide

### Disable Unused Services

**Review and disable services you don't use:**

**Navigation**: System → Settings → Administration

- **Disable HTTP redirect**: Consider disabling if you use custom port
- **Disable DNS rebind checks**: Only if necessary

---

## Part 8: Access Control Summary

### Recommended Configuration

**Web UI Access:**
- ✅ Custom admin user (not root)
- ✅ Root login disabled
- ✅ 2FA enabled
- ✅ Strong password (16+ characters)
- ✅ Floating rules restrict access to MGMT and Tailscale only
- ✅ HTTPS only (port 443 or custom)
- ✅ Accessible only from MGMT VLAN (firewall rules)

**SSH Access:**
- ✅ Custom admin user (not root)
- ✅ SSH key authentication
- ✅ Root SSH login disabled
- ✅ Password authentication disabled (after key setup)
- ✅ SSH only enabled when needed
- ✅ Consider custom SSH port (e.g., 2222)

**Console Access:**
- ✅ Root still available (for emergencies)
- ✅ Physical access required
- ✅ No remote access

---

## Part 9: Backup Admin Credentials

### Secure Credential Storage

**Store in password manager:**
- Admin username
- Admin password
- 2FA backup codes
- SSH private key passphrase
- Root password (for console emergencies)

**Recommended password managers:**
- 1Password
- Bitwarden
- KeePassXC

### Backup SSH Keys

```bash
# Backup private key securely
cp ~/.ssh/opnsense_admin_ed25519 /secure/backup/location/
cp ~/.ssh/opnsense_admin_ed25519.pub /secure/backup/location/

# Encrypt backup (optional)
gpg -c ~/.ssh/opnsense_admin_ed25519
```

---

## Part 10: Security Checklist

After implementing security hardening:

- [ ] Created dedicated admin user with strong password
- [ ] Tested admin user can login to web UI
- [ ] Tested admin user can SSH
- [ ] Added SSH public key to admin user
- [ ] Tested SSH key authentication works
- [ ] Disabled root web UI login
- [ ] Disabled root SSH login
- [ ] Disabled SSH password authentication (optional)
- [ ] Enabled 2FA for admin user
- [ ] Tested 2FA login works
- [ ] Saved 2FA backup codes securely
- [ ] Configured automatic security updates
- [ ] Restricted web UI access via firewall rules
- [ ] Backed up SSH keys and credentials
- [ ] Documented admin credentials in password manager
- [ ] Verified console access (root) still works for emergencies

---

## Part 11: Emergency Access

### If Locked Out of Web UI

**Method 1: Console Access (Root)**

1. Connect keyboard/monitor to router
2. Login with root account (still works on console)
3. Reset admin user password:
   ```bash
   # From console
   passwd your-admin-username
   ```

**Method 2: Reset to Factory Defaults**

1. Boot to OPNsense loader menu
2. Select "Reset to factory defaults"
3. Reconfigure from scratch

**Prevent lockout:**
- Always test admin user BEFORE disabling root
- Keep root password documented securely
- Test SSH keys before disabling password auth

---

## Part 12: Monitoring and Auditing

### Review Login Attempts

**Navigation**: System → Log Files → System

**Filter for authentication events:**
- Search: "authentication"
- Look for failed login attempts
- Investigate suspicious activity

### Review User Activity

**Navigation**: System → Log Files → System

**Filter by username:**
- Search: "username"
- Audit administrative actions

### Set Up Alerts (Optional)

**Navigation**: System → Monit → Settings

Configure alerts for:
- Failed login attempts
- Configuration changes
- Service failures

---

## Configuration Summary

### Minimal Security Configuration

```
✅ Admin user created (not root)
✅ Root web UI login: DISABLED
✅ Root SSH login: DISABLED
✅ Strong passwords for all users
✅ Firewall rules restrict management access
✅ Automatic security updates enabled
```

### Recommended Security Configuration

```
✅ All minimal configuration items
✅ SSH key authentication enabled
✅ SSH password authentication disabled
✅ 2FA enabled for all admin users
✅ Custom web UI port (optional)
✅ Logging and monitoring configured
✅ Regular security audits
```

### Maximum Security Configuration

```
✅ All recommended configuration items
✅ Management access only via Tailscale VPN
✅ Web UI not accessible from any local VLAN
✅ SSH only accessible via Tailscale
✅ Console access required for local admin
✅ Hardware security key for 2FA (YubiKey)
```

---

## Related Documentation

- [01_OPNSENSE_INSTALLATION.md](01_OPNSENSE_INSTALLATION.md) - Initial setup
- [08_TAILSCALE_SETUP.md](08_TAILSCALE_SETUP.md) - Secure remote access via VPN
- [09_TAILSCALE_HTTPS.md](09_TAILSCALE_HTTPS.md) - HTTPS certificates
