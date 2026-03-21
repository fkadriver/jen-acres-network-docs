# Tailscale Setup — Install, Subnet Routing & HTTPS Cert

## Overview

Tailscale provides encrypted remote access to all VLANs and serves as the primary (and
in most configurations, only) remote access path to OPNsense management. It also provides
the HTTPS certificate for the web UI.

**Complete after**: [01_OPNSENSE_INSTALLATION.md](01_OPNSENSE_INSTALLATION.md) (em3 break-glass must already exist)
**Complete before**: [03_VLAN_CONFIG.md](03_VLAN_CONFIG.md)

---

## Part 1: Install Tailscale Plugin on OPNsense

### Step 1 — Install via Firmware Plugins (26.1)

**Navigation**: System → Firmware → Plugins

Search for `os-tailscale` → click **+** to install → confirm.

After installation, the menu item **VPN → Tailscale** appears.

### Step 2 — Enable and Configure Tailscale

**Navigation**: VPN → Tailscale → Settings

| Setting | Value |
|---|---|
| Enable | ✓ |
| Advertise routes | `192.168.0.0/16` |
| Accept routes | ✓ |
| Automatically configure web GUI certificate | ✓ *(see Part 3)* |

Click **Save** → **Apply**.

### Step 3 — Authenticate

**Navigation**: VPN → Tailscale → Status → **Click authentication URL**

Follow the link to login.tailscale.com and authorize the device.

**Verify from SSH:**
```bash
ssh scott@192.168.1.1
tailscale status
# Should show: 100.x.x.x   opnsense   you@   linux   active; subnet router
```

### Step 4 — Approve Subnet Routes in Tailscale Admin

1. Go to `https://login.tailscale.com/admin/machines`
2. Find OPNsense → click **⋮** → **Edit route settings**
3. Enable `192.168.0.0/16` subnet route
4. Click **Save**

### Step 5 — Disable Key Expiry

On the same machine page → **Disable key expiry**.
Prevents authentication from expiring after 180 days.

### Step 6 — Assign Tag

Click **Edit tags** → add `tag:network` → Save.

---

## Part 2: Configure OPNsense Firewall for Tailscale

### Create Tailscale_Net Alias

**Navigation**: Firewall → Aliases → + Add

| Setting | Value |
|---|---|
| Name | `Tailscale_Net` |
| Type | Network(s) |
| Content | `100.64.0.0/10` |
| Description | Tailscale VPN network (CGNAT range) |

Save → Apply.

### Add Tailscale_Net to Management_Access Alias

**Navigation**: Firewall → Aliases → edit `Management_Access`

Add `Tailscale_Net` to the content alongside `MGMT_Net`.

> After completing Part 3 (HTTPS mandate), `MGMT_Net` will be **removed** from this alias,
> leaving only `Tailscale_Net`. Leave it for now until Part 3 is verified.

### Allow WAN → Tailscale UDP (if direct connections fail)

**Navigation**: Firewall → Rules → WAN → + Add

| Setting | Value |
|---|---|
| Action | Pass |
| Protocol | UDP |
| Source | any |
| Destination | WAN address |
| Destination port | 41641 |
| Description | Allow Tailscale direct connections |

Save → Apply. (Optional — Tailscale works via relay without this, but direct is faster.)

### Allow Tailscale Interface Traffic

**Navigation**: Firewall → Rules → Tailscale (tailscale0)

Add a pass-all rule:

| Setting | Value |
|---|---|
| Action | Pass |
| Protocol | any |
| Source | any |
| Destination | any |
| Description | Allow all traffic on Tailscale interface |

Save → Apply.

---

## Part 3: HTTPS — Tailscale Cert & Restrict Access to Tailscale-Only

> **Note**: TLS cert changes have **no effect** on Dnsmasq DNS or DHCP (ports 53/67/68).
> Only the web UI (port 443) is affected.

**Target state:**

| Access method | Result |
|---|---|
| `https://192.168.1.1` (LAN) | Blocked by firewall |
| `https://opnsense.warthog-royal.ts.net` (Tailscale) | Valid Let's Encrypt cert |
| Private key in XML config backups | None |

### Step 1 — Verify Tailscale cert is on disk

```bash
ssh scott@opnsense.warthog-royal.ts.net

ls -la /var/db/tailscale/certs/
# Expected: opnsense.warthog-royal.ts.net.crt  and  .key

openssl x509 -in /var/db/tailscale/certs/opnsense.warthog-royal.ts.net.crt \
  -noout -dates
# notAfter should be 30-90 days out
```

If missing: `tailscale cert opnsense.warthog-royal.ts.net`

### Step 2 — Enable plugin auto-cert (if not done in Part 1)

**Navigation**: VPN → Tailscale → Settings

- **Automatically configure web GUI certificate**: ✓

This points lighttpd directly at `/var/db/tailscale/certs/` — the private key never
enters the OPNsense config XML.

### Step 3 — Remove any manually-imported certs

**Navigation**: System → Trust → Certificates

Delete any cert that was manually imported. Before deleting, confirm the SSL Certificate
dropdown in System → Settings → Administration is set to the Tailscale plugin-managed cert.

```bash
service lighttpd restart
```

Open `https://opnsense.warthog-royal.ts.net` — expect a valid Let's Encrypt cert.

### Step 4 — Mandate Tailscale-only access (remove MGMT_Net from alias)

**Navigation**: Firewall → Aliases → `Management_Access`

Remove `MGMT_Net` from content. Leave only `Tailscale_Net`.

Save → Apply Changes.

> **⚠️ After this step**, `https://192.168.1.1` is blocked by the floating rules.
> Verify `https://opnsense.warthog-royal.ts.net` works before applying.
> If locked out, use em3 break-glass (`https://192.168.99.1`).

**Test:**
```bash
# From a LAN device — should be blocked
curl -sk https://192.168.1.1 -o /dev/null -w "%{http_code}"

# From Tailscale — should work
open https://opnsense.warthog-royal.ts.net
```

### Certificate Auto-Renewal

Tailscale certs are valid 90 days and auto-renew at 60 days via the plugin. No manual
steps needed. Verify periodically:

```bash
openssl x509 -in /var/db/tailscale/certs/opnsense.warthog-royal.ts.net.crt \
  -noout -dates
# notAfter should always be 30-90 days ahead
```

If expired: `tailscale cert --force opnsense.warthog-royal.ts.net && service lighttpd restart`

---

## Part 4: Tailscale Admin Configuration

### MagicDNS

In Tailscale admin console → **DNS → Settings**:
- ✓ Enable MagicDNS
- Global nameserver: `192.168.10.10` (Pi-hole — configure after Part 6)

### ACL Policy

ACLs are managed in the `tailnet/` submodule (`tailnet/policy.hujson`).
Changes committed and pushed deploy via GitHub Actions.

Key rules:
- `group:admin` → full access to everything
- All members → access Secure_Net (192.168.0.0/20); Guest/IoT (192.168.16.0/20) intentionally blocked
- Tagged devices → full mesh (tagged-to-tagged)
- `tag:mgmt-admin` → SSH and HTTPS to network devices
- **SSH**: admin check-mode (12h re-auth) to tagged infra; members SSH to own devices only
- See [tailnet/README.md](../tailnet/README.md) for full SSH and ACL rule documentation

### Recommended Device Tags

| Device | Tags |
|--------|------|
| OPNsense | `tag:network` |
| Pi-hole Primary/Backup | `tag:server` |
| NAS01 | `tag:nas`, `tag:server` |
| Admin Laptop | `tag:mgmt-admin` |

---

## Part 5: Install Tailscale on Other Devices

Pi-holes and other Linux devices:
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --accept-routes --ssh
# Do NOT use --accept-dns (Pi-hole IS the DNS server)
```

---

## Verification Checklist

```
[ ] Tailscale status shows: active; subnet router
[ ] 192.168.0.0/16 subnet routes approved in Tailscale admin
[ ] opnsense.warthog-royal.ts.net reachable with valid cert
[ ] 192.168.1.1 direct access blocked (Tailscale-only mandate in effect)
[ ] Management_Access alias contains only Tailscale_Net
[ ] em3 break-glass (192.168.99.1) still accessible — unaffected
[ ] SSH works: ssh scott@opnsense.warthog-royal.ts.net
[ ] Key expiry disabled on OPNsense device in Tailscale admin
[ ] Device tagged as tag:network
```

---

## Troubleshooting

**Tailscale not starting after reboot:**
```bash
service tailscaled status
service tailscaled start
```

**Cannot reach subnets from remote:**
- Check subnet route approval in Tailscale admin (login.tailscale.com/admin/machines)
- Check `Tailscale_Net` alias = `100.64.0.0/10` in Firewall → Aliases
- Check tailscale0 interface pass-all rule exists

**High latency (relay instead of direct):**
- Add WAN UDP 41641 pass rule (Part 2)
- Check if ISP uses CGNAT blocking P2P

**Authentication expired:**
```bash
tailscale up --advertise-routes=192.168.0.0/16 --accept-routes
```
Then disable key expiry in admin console.

**Cert expired or missing:**
```bash
tailscale cert --force opnsense.warthog-royal.ts.net
service lighttpd restart
```
