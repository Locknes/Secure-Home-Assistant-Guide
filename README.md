# Secure Home Assistant Remote Access Guide
### Cloudflare Edge + UniFi Firewall + Nginx Proxy Manager (NPM)

A complete, beginner-friendly walkthrough for securely exposing Home Assistant to the internet. This setup conceals your home IP address, silently drops port scanners at your router firewall, provides end-to-end SSL encryption, and ensures rock-solid dashboard WebSocket connections.

---

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Before You Start (Prerequisites)](#before-you-start-prerequisites)
3. [Step 0: Set Static Local IPs (DHCP Reservations)](#step-0-set-static-local-ips-dhcp-reservations)
4. [Step 1: Cloudflare Edge Setup](#step-1-cloudflare-edge-setup)
5. [Step 2: UniFi Gateway Firewall Rules](#step-2-unifi-gateway-firewall-rules)
6. [Step 3: Nginx Proxy Manager (NPM) Setup](#step-3-nginx-proxy-manager-npm-setup)
7. [Step 4: Home Assistant Configuration](#step-4-home-assistant-configuration)
8. [Step 5: Verification (Avoiding the NAT Loopback Trap)](#step-5-verification-avoiding-the-nat-loopback-trap)
9. [Secure Remote Troubleshooting & Maintenance](#secure-remote-troubleshooting--maintenance)
10. [Helpful Integrations & DynDNS](#helpful-integrations--dyndns)
11. [Troubleshooting Search Terms](#troubleshooting-search-terms)
12. [Documentation & Reference Links](#documentation--reference-links)

---

## Architecture Overview

```text
[Remote Phone App / Laptop]
           │  (Visits https://ha.yourdomain.com)
           ▼
[Cloudflare Edge Proxy]   <-- Hides home IP, handles public SSL, WAF & DDoS
           │  (Proxies traffic using Cloudflare Anycast IP pool)
           ▼
[UniFi Gateway Firewall]  <-- Internet In: ONLY accepts Cloudflare IPs; drops direct WAN scans
           │  (Port forwards 443 to internal reverse proxy)
           ▼
[Nginx Proxy Manager]     <-- Terminates 15-year Origin SSL, enables WebSockets & sets Real-IP
           │  (Proxies to port 8123)
           ▼
[Home Assistant Core]     <-- trusted_proxies enabled, serves dashboard & webhooks
```

### Why This Security Model Works:
1. **Your Residential IP is Shielded:** Public DNS queries return Cloudflare Anycast IP addresses, never your actual home IP.
2. **Direct WAN Port Scanners Are Dropped:** If an internet scanner or botnet (Shodan, Censys) probes your home IP directly on port 443, your UniFi gateway drops the packets before they reach your reverse proxy.
3. **End-to-End Encryption:** Strict SSL encryption is enforced between client and Cloudflare, and between Cloudflare and your Nginx reverse proxy using a dedicated 15-year Origin Certificate.
4. **WebSocket & Dashboard Stability:** Caching is bypassed for dynamic state changes, preventing dashboard freezes or disconnect banners.

---

## Before You Start (Prerequisites)

Before beginning, ensure you have:
* **A Custom Domain Name:** Purchased through any registrar (Cloudflare Registrar, Namecheap, Porkbun, etc.).
* **Cloudflare Managing Your Domain:** Your domain's nameservers must be pointed to Cloudflare.
* **Home Assistant Installed:** Running Home Assistant OS, Supervised, or Container on your local network.
* **Nginx Proxy Manager (NPM):**
  * *Option A (Easiest — Home Assistant OS):* Installed directly from the **Add-on Store** (`Nginx Proxy Manager`).
  * *Option B (Standalone):* Running in Docker, an LXC container, or a virtual machine on your network.

---

## Step 0: Set Static Local IPs (DHCP Reservations)

> [!IMPORTANT]
> **Why this is critical:** Routers assign dynamic local IP addresses that can change when devices reboot. If your Home Assistant or Nginx machine gets a new IP, all port forwarding rules and proxy host targets will immediately break.

1. Open your **UniFi Network** dashboard.
2. Navigate to **Client Devices** and find your Home Assistant server (and your Nginx host if running separately).
3. Click the device to open its settings drawer on the right.
4. Select the **Settings** tab (or **IP Settings**).
5. Check **Use Fixed IP Address**.
6. Note the assigned address:
   * Example Nginx Proxy Manager IP: `192.168.1.50`
   * Example Home Assistant IP: `192.168.1.50` (if running NPM as an HA add-on) or `192.168.1.100` (if separate).
7. Click **Apply Changes**.

---

## Step 1: Cloudflare Edge Setup

### 1. Create the DNS Record
1. In Cloudflare, go to **DNS ➔ Records ➔ Add Record**.
2. **Type:** `A`
3. **Name:** `ha` (or your preferred subdomain)
4. **IPv4 address:** Your current public WAN IP (find it at [whatismyip.com](https://www.whatismyip.com/)).
5. **Proxy status:** **Proxied (Orange Cloud ON)** *(Essential!)*.
6. Click **Save**.

> [!TIP]
> **Automating Dynamic IP (DynDNS):**
> If your ISP changes your public WAN IP, don't update it manually. Once your system is up, install the [Home Assistant Cloudflare Integration](https://www.home-assistant.io/integrations/cloudflare/). Home Assistant will automatically update your Cloudflare `A` record whenever your WAN IP changes while preserving the **Proxied** status.

### 2. Configure Strict SSL Mode
1. In Cloudflare, navigate to **SSL/TLS ➔ Overview**.
2. Select **Full (strict)** encryption mode.

### 3. Generate a 15-Year Origin Certificate
1. Go to **SSL/TLS ➔ Origin Server ➔ Create Certificate**.
2. Leave the defaults (RSA 2048, 15-year validity, covering `yourdomain.com` and `*.yourdomain.com`).
3. Click **Create**.
4. Copy the **Origin Certificate** and save it on your computer as `origin.pem`.
5. Copy the **Private Key** and save it as `privkey.pem`.
*(You will import these into Nginx Proxy Manager in Step 3).*

### 4. Enable WebSockets & Bypass Caching
1. Navigate to **Network** and toggle **WebSockets** **ON**.
2. Navigate to **Caching ➔ Cache Rules ➔ Create Rule**:
   * **Rule Name:** `Bypass HA Cache`
   * **Field:** `Hostname` | **Operator:** `equals` | **Value:** `ha.yourdomain.com`
   * **Cache Eligibility:** **Bypass cache**
   * Click **Deploy**.
3. Under **Speed ➔ Optimization** (or Page Rules), ensure **Rocket Loader** and **Auto Minify** are turned **OFF** for your Home Assistant subdomain, as they break Lovelace dashboard JavaScript.

### 5. (Optional) WAF Geo-Blocking Rule
To block access outside your home country while still allowing mobile app webhooks:
1. Navigate to **Security ➔ WAF ➔ Custom Rules ➔ Create Rule**.
2. **Expression:**
   ```text
   (http.host eq "ha.yourdomain.com" and ip.geoip.country ne "US" and not http.request.uri.path starts_with "/api/webhook/")
   ```
3. **Action:** **Block** (or **Managed Challenge**).
4. Click **Deploy**.

---

## Step 2: UniFi Gateway Firewall Rules

This step stops direct port scanning by ensuring only Cloudflare proxies can reach port 443.

### 1. Create Firewall IP Groups in UniFi
In UniFi Network, navigate to **Settings ➔ Security ➔ Firewall / IP Groups** (or **Profiles ➔ IP Groups**):

1. **`Cloudflare-IPv4`** (Type: `IPv4 Address/Subnet`):
   * Add Cloudflare's published CIDR blocks from [cloudflare.com/ips-v4](https://www.cloudflare.com/ips-v4).
2. **`Cloudflare-IPv6`** (Type: `IPv6 Address/Subnet`):
   * Add Cloudflare's published CIDR blocks from [cloudflare.com/ips-v6](https://www.cloudflare.com/ips-v6).
3. **`IPs - Reverse Proxy`** (Type: `IPv4 Address/Subnet`):
   * Enter the fixed IP of your Nginx Proxy Manager (e.g., `192.168.1.50`).
4. **`Port - HTTPS`** (Type: `Port Group`):
   * Port: `443`

### 2. Configure Port Forwarding
Navigate to **Settings ➔ Security ➔ Port Forwarding ➔ Create New**:
* **Name:** `Nginx HTTPS`
* **From:** `WAN` (or `Both`)
* **Port:** `443`
* **Forward IP:** `192.168.1.50` (Your Nginx host local IP)
* **Forward Port:** `443`
* **Protocol:** `TCP`
* Click **Apply Changes**.

### 3. Add `Internet In` (WAN IN) Firewall Rules
Navigate to **Settings ➔ Security ➔ Firewall Rules ➔ Internet In**.

Create the following rules **in this exact sequence**:

1. **Rule 1 — Allow Established & Related:**
   * **Action:** `Accept`
   * **Protocol:** `All`
   * **State:** Check `Established` and `Related`

2. **Rule 2 — Allow Cloudflare Traffic Only:**
   * **Action:** `Accept`
   * **Protocol:** `TCP`
   * **Source:** Select Port/IP Group ➔ `Cloudflare-IPv4`
   * **Destination:** Select Port/IP Group ➔ `IPs - Reverse Proxy`, Port `Port - HTTPS`

3. **Rule 3 — Drop All Other Inbound 443 Scans:**
   * **Action:** `Drop`
   * **Protocol:** `TCP`
   * **Source:** `Any`
   * **Destination:** Select Port/IP Group ➔ `IPs - Reverse Proxy`, Port `Port - HTTPS`

> [!IMPORTANT]
> Rules execute top-down in UniFi. **Rule 2 (Allow Cloudflare)** must sit above **Rule 3 (Drop All)**. Any scanner attempting to hit your home IP directly on port 443 will be silently dropped.

---

## Step 3: Nginx Proxy Manager (NPM) Setup

### 1. Import the Cloudflare Origin Certificate
1. Open the Nginx Proxy Manager web UI.
2. Navigate to **SSL Certificates ➔ Add SSL Certificate ➔ Custom**.
3. **Name:** `Cloudflare Origin Cert`
4. **Certificate Key:** Paste the complete text of `privkey.pem`.
5. **Certificate:** Paste the complete text of `origin.pem`.
6. Click **Save**.

### 2. Create the Proxy Host
Navigate to **Hosts ➔ Proxy Hosts ➔ Add Proxy Host**:

* **Details Tab:**
  * **Domain Names:** `ha.yourdomain.com`
  * **Scheme:** `http`
  * **Forward Hostname / IP:** Local IP of your Home Assistant server (use `127.0.0.1` if NPM runs as an HA add-on; otherwise use `192.168.1.100`)
  * **Forward Port:** `8123`
  * **Cache Assets:** `OFF`
  * **Block Common Exploits:** `ON`
  * **Websockets Support:** **ON** *(Mandatory — live dashboard updates and media controls require WebSockets)*
* **SSL Tab:**
  * **SSL Certificate:** Select `Cloudflare Origin Cert`
  * **Force SSL:** `ON`
  * **HTTP/2 Support:** `ON`
  * **HSTS Enabled:** `ON`
* Click **Save**.

### 3. Restore Visitor Real IPs (Optional but Recommended)
Under the proxy host's **Advanced** tab, add Nginx real-ip directives so Home Assistant log files and brute-force protection see the visitor's real IP instead of Cloudflare's proxy address:

```nginx
real_ip_header CF-Connecting-IP;
set_real_ip_from 103.21.244.0/22;
set_real_ip_from 103.22.200.0/22;
set_real_ip_from 103.31.4.0/22;
set_real_ip_from 104.16.0.0/13;
set_real_ip_from 104.24.0.0/14;
set_real_ip_from 108.162.192.0/18;
set_real_ip_from 131.0.72.0/22;
set_real_ip_from 141.101.64.0/18;
set_real_ip_from 162.158.0.0/15;
set_real_ip_from 172.64.0.0/13;
set_real_ip_from 173.245.48.0/20;
set_real_ip_from 188.114.96.0/20;
set_real_ip_from 190.93.240.0/20;
set_real_ip_from 197.234.240.0/22;
set_real_ip_from 198.41.128.0/17;
```

---

## Step 4: Home Assistant Configuration

Home Assistant blocks reverse proxy traffic with `HTTP 400: Bad Request` unless explicitly configured to trust the proxy.

### 1. Update `configuration.yaml`
Add or update the `http` block:

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 127.0.0.1
    - 192.168.1.50      # Replace with your Nginx Proxy Manager local IP
    - 172.30.33.0/24    # Include this if NPM is running as an official Home Assistant OS add-on
  ip_ban_enabled: true
  login_attempts_threshold: 5
```

### 2. Lock Down Sensitive Local Webhooks
If you have admin automations or internal notification webhooks that should never be accessible from the internet, add `local_only: true` to the webhook trigger:

```yaml
trigger:
  - platform: webhook
    webhook_id: "your_private_webhook_id"
    local_only: true
```
Home Assistant will automatically drop requests arriving via Cloudflare or external proxies, only allowing local RFC1918 subnets to trigger the automation.

### 3. Restart Home Assistant
Go to **Developer Tools ➔ YAML ➔ Restart** to apply changes.

---

## Step 5: Verification (Avoiding the NAT Loopback Trap)

> [!WARNING]
> **The NAT Loopback Trap:**
> If you test `https://ha.yourdomain.com` from a PC connected to your home Wi-Fi, your router might block the hairpin connection or loop back locally, giving you a false failure.

### The Foolproof Test:
1. **Turn OFF Wi-Fi on your smartphone** so it is strictly using cellular data (5G/LTE).
2. Open a mobile browser and visit `https://ha.yourdomain.com`.
3. You should see your Home Assistant login screen with a secure padlock.
4. Log in and toggle a light or switch. It should react instantly with no "reconnecting" banner.
5. In Home Assistant, go to **Settings ➔ System ➔ Logs**. Verify there are no `A request from a reverse proxy was received from ...` errors.

---

## Secure Remote Troubleshooting & Maintenance

If you need to access or fix your setup remotely—or invite a trusted friend/technician to help troubleshoot when things break—follow these best practices to do it safely:

### 1. The Cardinal Rule: Never Expose Admin Ports
* **Never port-forward port 81 (NPM Admin UI)** to the internet.
* **Never port-forward port 22 (SSH)** to the internet.
* **Never port-forward port 8443 / 443 (UniFi Gateway Console)** to the internet.
Only the public reverse proxy port (`443`) should be forwarded.

### 2. Best Method for Deep Troubleshooting: Mesh VPN (Tailscale)
If Home Assistant hangs, a bad YAML edit breaks boot, or Nginx stops routing traffic, the public reverse proxy will be unreachable. You need an out-of-band "break-glass" path:
* **Install Tailscale on the Host:** Install Tailscale on the Home Assistant machine (or your underlying Docker/Proxmox server).
* **Node Sharing:** You can securely "share" just the Home Assistant node with a helper's Tailscale account.
  * They get an encrypted tunnel directly to the machine (`100.x.x.x`).
  * They can access SSH and the local web UI (`http://100.x.x.x:8123` or NPM on `:81`).
  * They **cannot** access the rest of your home network.
  * You can revoke their access in one click when finished.

### 3. Best Method for Web-Only Remote Access: Cloudflare Zero Trust (Access)
If you want to allow remote web admin access without requiring VPN software:
1. In Cloudflare, navigate to **Zero Trust ➔ Access ➔ Applications**.
2. Add an application protecting `ha.yourdomain.com` (or a dedicated maintenance subdomain like `npm.yourdomain.com`).
3. Set an **Access Policy** requiring **One-Time Email PIN** or **Google/GitHub OAuth**.
4. Whitelist only specific email addresses (e.g., your email and your helper's email).
5. Anyone visiting the URL must enter a temporary code emailed to them before Cloudflare connects them to your home server.

### 4. Temporary Admin Accounts in Home Assistant
When inviting someone to fix something in Home Assistant:
* **Never give out your primary credentials.**
* Go to **Settings ➔ People ➔ Users ➔ Add User**.
* Create a dedicated user with **Administrator** rights.
* Have them set up **Two-Factor Authentication (TOTP)** on first login.
* Once the issue is resolved, delete or disable the user account immediately.

---

## Helpful Integrations & DynDNS

1. **[Cloudflare Integration](https://www.home-assistant.io/integrations/cloudflare/):** Automatically updates your Cloudflare `A` record when your ISP changes your public WAN IP while maintaining proxy protection.
2. **[UniFi Network Integration](https://www.home-assistant.io/integrations/unifi/):** Connects to your gateway to monitor router health, WAN uptime, port metrics, and network client presence.
3. **[Certificate Expiry Sensor](https://www.home-assistant.io/integrations/cert_expiry/):** Native sensor that monitors `ha.yourdomain.com` and alerts you weeks before any SSL certificate renewal issue.
4. **[Nginx Proxy Manager Add-on](https://github.com/hassio-addons/addon-nginx-proxy-manager):** Runs NPM directly inside Home Assistant OS, keeping reverse proxy configuration in regular Home Assistant backups.
5. **[Tailscale](https://www.home-assistant.io/integrations/tailscale/) / [WireGuard](https://www.home-assistant.io/integrations/wireguard/):** Provides an encrypted backdoor into your network if Cloudflare or public DNS ever has an issue.

---

## Troubleshooting Search Terms

* `"Home Assistant" "trusted_proxies" 400 Bad Request`
* `"Cloudflare" "Home Assistant" websocket disconnection 1006`
* `"Cloudflare" origin CA certificate nginx proxy manager 15 year`
* `"Cloudflare" cache rules bypass home assistant dashboard`
* `UniFi firewall rule allow only cloudflare IPs port forward`
* `UniFi drop direct WAN port forward traffic cloudflare bypass`
* `Nginx Proxy Manager "real_ip_header CF-Connecting-IP" set_real_ip_from`
* `Tailscale Home Assistant node sharing remote fix`

---

## Documentation & Reference Links

* **Inspiration Article:** [Matthew Hodgkins — Securing Home Assistant with Cloudflare](https://hodgkins.io/blog/securing-home-assitant-with-cloudflare/)
* **IP Ranges:** [Cloudflare Official Public IP List](https://www.cloudflare.com/ips/)
* **Origin CA Setup:** [Cloudflare Origin CA Certificate Documentation](https://developers.cloudflare.com/ssl/origin-configuration/origin-ca/)
* **Home Assistant HTTP:** [Home Assistant HTTP Integration Documentation](https://www.home-assistant.io/integrations/http/)
* **UniFi Firewall:** [Ubiquiti UniFi Gateway Introduction to Firewall Rules](https://help.ui.com/hc/en-us/articles/115003173168-UniFi-Gateway-Introduction-to-Firewall-Rules)
* **Nginx Proxy Manager:** [Nginx Proxy Manager Official Documentation](https://nginxproxymanager.com/)
* **Cloudflare Access:** [Cloudflare Zero Trust Access Documentation](https://developers.cloudflare.com/cloudflare-one/policies/access/)
* **Tailscale Node Sharing:** [Tailscale Node Sharing Documentation](https://tailscale.com/kb/1084/sharing)
