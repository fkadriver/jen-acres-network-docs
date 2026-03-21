# Tailscale HTTPS Certificates for OPNsense and Pi-hole

## Overview

Tailscale provides free, automatically-renewed TLS certificates for devices on your tailnet through **Tailscale HTTPS**. This eliminates self-signed certificate warnings when accessing:
- OPNsense web interface
- Pi-hole admin panel
- Any other web services on your network

## Benefits

✅ **Valid TLS Certificates**: Issued by Let's Encrypt via Tailscale
✅ **No Browser Warnings**: No more "Your connection is not private" errors
✅ **Automatic Renewal**: Certificates renew automatically
✅ **Zero Configuration**: Works with MagicDNS hostnames
✅ **No Port Forwarding**: All traffic stays within Tailscale VPN
✅ **Free**: No cost, included with Tailscale

## Prerequisites

- [ ] Tailscale installed on OPNsense (see [08_TAILSCALE_SETUP.md](08_TAILSCALE_SETUP.md))
- [ ] MagicDNS enabled in Tailscale
- [ ] Devices have Tailscale hostnames

---

## Part 1: Enable Tailscale HTTPS

### Step 1: Enable MagicDNS and HTTPS

**Tailscale Admin Console**: https://login.tailscale.com/admin/dns

1. **Enable MagicDNS**:
   - ✓ Enable MagicDNS
   - This assigns `<hostname>.your-tailnet.ts.net` names to all devices

2. **Enable HTTPS**:
   - ✓ Enable HTTPS
   - This allows Tailscale to provision TLS certificates

Click **Save**

### Step 2: Verify Tailscale Hostnames

From any device on your Tailscale network:

```bash
tailscale status

# You should see hostnames like:
# 100.x.x.x   opnsense-router.your-tailnet.ts.net   you@   linux   -
# 100.y.y.y   pihole.your-tailnet.ts.net            you@   linux   -
```

**Note**: Your actual domain will be `your-tailnet.ts.net` where `your-tailnet` is based on your Tailscale account.

---

## Part 2: Configure OPNsense to Use Tailscale Certificates

### Method 1: Using Tailscale's Built-in Certificate Server (Recommended)

Tailscale can serve HTTPS on port 443 for the OPNsense machine name.

**On OPNsense via SSH:**

```bash
# Enable HTTPS certificate serving
tailscale cert opnsense-router.your-tailnet.ts.net

# This creates certificate files:
# /var/db/tailscale/certs/opnsense-router.your-tailnet.ts.net.crt
# /var/db/tailscale/certs/opnsense-router.your-tailnet.ts.net.key
```

**Note**: Replace `opnsense-router.your-tailnet.ts.net` with your actual Tailscale hostname from `tailscale status`.

### Configure OPNsense Web Interface

**Via OPNsense Web UI:**

**Navigation**: System → Settings → Administration

1. **Protocol**: HTTPS
2. **SSL Certificate**: Custom certificate (we'll import Tailscale cert)
3. **TCP Port**: 443 (or keep as-is if you prefer another port)

**Import Tailscale Certificate:**

**Navigation**: System → Trust → Certificates

**Import Certificate:**
1. Click **+ Add**
2. **Method**: Import an existing Certificate
3. **Descriptive name**: Tailscale HTTPS Certificate
4. **Certificate data**:
   ```bash
   # On OPNsense, get cert content:
   cat /var/db/tailscale/certs/opnsense-router.your-tailnet.ts.net.crt
   ```
   Copy and paste the output
5. **Private key data**:
   ```bash
   # On OPNsense, get key content:
   cat /var/db/tailscale/certs/opnsense-router.your-tailnet.ts.net.key
   ```
   Copy and paste the output
6. Click **Save**

**Assign Certificate to Web Interface:**

**Navigation**: System → Settings → Administration

- **SSL Certificate**: Select "Tailscale HTTPS Certificate"
- Click **Save**

**Restart Web Server:**
```bash
# Via SSH on OPNsense
service lighttpd restart
```

### Access OPNsense with Valid Certificate

From any device on your Tailscale network:

```
https://opnsense-router.your-tailnet.ts.net
```

✅ **No certificate warnings!**

---

## Part 3: Configure Pi-hole to Use Tailscale Certificates

### Enable HTTPS on Pi-hole

**On Raspberry Pi via SSH:**

```bash
# Get Tailscale certificate
sudo tailscale cert pihole.your-tailnet.ts.net

# Certificates saved to:
# /var/lib/tailscale/certs/pihole.your-tailnet.ts.net.crt
# /var/lib/tailscale/certs/pihole.your-tailnet.ts.net.key
```

### Configure Lighttpd for HTTPS

**Edit lighttpd SSL configuration:**

```bash
sudo nano /etc/lighttpd/external.conf
```

**Add:**
```
# Enable SSL
server.modules += ( "mod_openssl" )

# SSL Configuration
$SERVER["socket"] == ":443" {
    ssl.engine = "enable"
    ssl.pemfile = "/var/lib/tailscale/certs/pihole.your-tailnet.ts.net.combined.pem"
    ssl.ca-file = "/var/lib/tailscale/certs/pihole.your-tailnet.ts.net.crt"
}

# Redirect HTTP to HTTPS (optional)
$HTTP["scheme"] == "http" {
    $HTTP["host"] =~ ".*" {
        url.redirect = (".*" => "https://%0$0")
    }
}
```

**Create combined PEM file** (lighttpd requires cert + key in one file):

```bash
sudo cat /var/lib/tailscale/certs/pihole.your-tailnet.ts.net.crt \
         /var/lib/tailscale/certs/pihole.your-tailnet.ts.net.key | \
sudo tee /var/lib/tailscale/certs/pihole.your-tailnet.ts.net.combined.pem

# Set permissions
sudo chmod 600 /var/lib/tailscale/certs/pihole.your-tailnet.ts.net.combined.pem
sudo chown www-data:www-data /var/lib/tailscale/certs/pihole.your-tailnet.ts.net.combined.pem
```

**Restart lighttpd:**

```bash
sudo service lighttpd restart
```

### Access Pi-hole with Valid Certificate

From any device on your Tailscale network:

```
https://pihole.your-tailnet.ts.net/admin
```

✅ **Valid certificate, no warnings!**

---

## Part 4: Automatic Certificate Renewal

Tailscale certificates are valid for **90 days** and auto-renew at **60 days**.

### Create Renewal Script for Pi-hole

Since lighttpd needs the combined PEM file, create a renewal script:

```bash
sudo nano /usr/local/bin/tailscale-cert-renew.sh
```

**Add:**
```bash
#!/bin/bash
# Tailscale Certificate Renewal for Pi-hole

HOSTNAME="pihole.your-tailnet.ts.net"
CERT_DIR="/var/lib/tailscale/certs"

# Request new certificate
tailscale cert $HOSTNAME

# Recreate combined PEM
cat $CERT_DIR/$HOSTNAME.crt $CERT_DIR/$HOSTNAME.key > $CERT_DIR/$HOSTNAME.combined.pem
chmod 600 $CERT_DIR/$HOSTNAME.combined.pem
chown www-data:www-data $CERT_DIR/$HOSTNAME.combined.pem

# Reload lighttpd
service lighttpd reload

echo "$(date): Certificate renewed for $HOSTNAME"
```

**Make executable:**
```bash
sudo chmod +x /usr/local/bin/tailscale-cert-renew.sh
```

**Add to cron** (run monthly):
```bash
sudo crontab -e
```

**Add:**
```
# Renew Tailscale certificate monthly (runs on 1st of each month)
0 3 1 * * /usr/local/bin/tailscale-cert-renew.sh >> /var/log/tailscale-cert-renew.log 2>&1
```

### OPNsense Certificate Renewal

For OPNsense, you'll need to manually re-import the certificate when it renews (every 90 days), or create a script to automate this:

```bash
#!/bin/sh
# /usr/local/bin/tailscale-cert-renew.sh

HOSTNAME="opnsense-router.your-tailnet.ts.net"
CERT_DIR="/var/db/tailscale/certs"

# Request renewal
tailscale cert $HOSTNAME

# Note: You'll need to re-import via Web UI or use OPNsense API
echo "Certificate renewed. Re-import via System → Trust → Certificates"
```

---

## Part 5: Alternative - Tailscale Serve (Simplest Option)

Tailscale has a built-in reverse proxy called **Tailscale Serve** that handles HTTPS automatically without manual certificate management.

### For Simple Web Services

**On Pi-hole (if running web service on port 80):**

```bash
# Serve Pi-hole on HTTPS via Tailscale
sudo tailscale serve --bg http://localhost:80
```

This:
- Serves `https://pihole.your-tailnet.ts.net`
- Automatically provisions and renews certificates
- Proxies to local Pi-hole on port 80
- No lighttpd configuration needed!

**On OPNsense:**

```bash
# Serve OPNsense web UI on HTTPS
tailscale serve --bg https://localhost:443
```

Access via: `https://opnsense-router.your-tailnet.ts.net`

**Advantages:**
- ✅ Zero certificate management
- ✅ Automatic HTTPS
- ✅ Automatic renewal
- ✅ Simple one-command setup

**Disadvantages:**
- ⚠️ Only accessible via Tailscale (not on local network)
- ⚠️ Adds slight overhead (extra proxy hop)

---

## Part 6: Verification and Testing

### Test Certificate Validity

**From any device on Tailscale network:**

```bash
# Check OPNsense certificate
openssl s_client -connect opnsense-router.your-tailnet.ts.net:443 -showcerts

# Should show:
# - Issuer: Let's Encrypt
# - Valid dates
# - No self-signed warnings

# Check Pi-hole certificate
openssl s_client -connect pihole.your-tailnet.ts.net:443 -showcerts
```

### Browser Test

Open in browser:
- `https://opnsense-router.your-tailnet.ts.net`
- `https://pihole.your-tailnet.ts.net/admin`

**Expected:**
- ✅ Padlock icon (secure)
- ✅ Valid certificate from Let's Encrypt
- ✅ No warnings

---

## Part 7: Troubleshooting

### Certificate Not Working

**Check Tailscale status:**
```bash
tailscale status
# Verify HTTPS is enabled
```

**Re-request certificate:**
```bash
# On the device
sudo tailscale cert <hostname>.your-tailnet.ts.net --force
```

### "Certificate is for wrong domain"

**Ensure you're using the full Tailscale hostname:**
- ❌ Wrong: `https://opnsense-router`
- ❌ Wrong: `https://192.168.1.1`
- ✅ Correct: `https://opnsense-router.your-tailnet.ts.net`

### MagicDNS Not Resolving

**Check MagicDNS settings:**
- Tailscale admin console → DNS
- Verify "Enable MagicDNS" is checked
- Verify "Enable HTTPS" is checked

**On client device:**
```bash
nslookup opnsense-router.your-tailnet.ts.net
# Should resolve to Tailscale IP (100.x.x.x)
```

### Certificate Files Not Found

**Check certificate location:**

**On Linux (Pi-hole):**
```bash
ls -la /var/lib/tailscale/certs/
```

**On FreeBSD (OPNsense):**
```bash
ls -la /var/db/tailscale/certs/
```

**If missing, request certificate:**
```bash
tailscale cert <hostname>.your-tailnet.ts.net
```

---

## Part 8: Best Practices

### Security Recommendations

1. **Keep Certificates Private**: Don't share private key files
2. **Use Strong ACLs**: Limit who can access your subnet via Tailscale ACLs
3. **Monitor Expiry**: Set up monitoring for certificate expiration
4. **Backup Certificates**: Include in your backup routine (though they can be re-issued)

### Performance Considerations

1. **Tailscale Serve vs Direct**:
   - Tailscale Serve adds ~1ms latency (negligible for web UIs)
   - Direct certificate installation is slightly faster

2. **Certificate Caching**:
   - Browsers cache Tailscale certificates normally
   - No performance impact after initial load

### Recommended Setup

**For OPNsense**: Manual certificate import (Method 1)
- Provides direct HTTPS access
- Works on local network and Tailscale
- Requires manual renewal every 90 days

**For Pi-hole**: Tailscale Serve (Part 5)
- Simplest setup
- Automatic certificate management
- Sufficient for admin panel access

---

## Configuration Summary

### OPNsense with Tailscale Certificates

```bash
# 1. Request certificate
tailscale cert opnsense-router.your-tailnet.ts.net

# 2. Import via Web UI: System → Trust → Certificates

# 3. Assign to Web UI: System → Settings → Administration

# 4. Access via:
https://opnsense-router.your-tailnet.ts.net
```

### Pi-hole with Tailscale Serve (Recommended)

```bash
# One command setup:
sudo tailscale serve --bg http://localhost:80

# Access via:
https://pihole.your-tailnet.ts.net/admin
```

### Pi-hole with Manual Certificate (Advanced)

```bash
# 1. Request certificate
sudo tailscale cert pihole.your-tailnet.ts.net

# 2. Create combined PEM
sudo cat /var/lib/tailscale/certs/pihole.*.crt \
         /var/lib/tailscale/certs/pihole.*.key > \
         /var/lib/tailscale/certs/pihole.*.combined.pem

# 3. Configure lighttpd (see Part 3)

# 4. Access via:
https://pihole.your-tailnet.ts.net/admin
```

---

## Related Documentation

- [08_TAILSCALE_SETUP.md](08_TAILSCALE_SETUP.md) - Tailscale subnet router setup
- [05_OPNSENSE_PIHOLE_INTEGRATION.md](05_OPNSENSE_PIHOLE_INTEGRATION.md) - Pi-hole integration
- Official Tailscale HTTPS docs: https://tailscale.com/kb/1153/enabling-https/
