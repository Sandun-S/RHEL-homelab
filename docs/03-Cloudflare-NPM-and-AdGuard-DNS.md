# 03 — Cloudflare Zero Trust, Client SSH Setup, Nginx Proxy Manager & AdGuard Split-DNS

This guide covers how to connect to the home server from anywhere or on the local LAN, how to install `cloudflared` on both the **RHEL server** and **your client PC**, how to set up Cloudflare Zero Trust Tunnels, and how to configure Nginx Proxy Manager (NPM) and AdGuard Home Split-DNS.

---

## 1. Host Setup: Installing `cloudflared` on RHEL 10.1 (`syntax`)

### 1.1 Add Official Cloudflare RPM Repository & Install `cloudflared`
On `syntax` (`192.168.1.3`), `cloudflared` is installed from Cloudflare's official RPM repository:

```bash
sudo tee /etc/yum.repos.d/cloudflared.repo > /dev/null << 'EOF'
[cloudflared-stable]
name=cloudflared-stable
baseurl=https://pkg.cloudflare.com/cloudflared/rpm
enabled=1
type=rpm
gpgcheck=1
gpgkey=https://pkg.cloudflare.com/cloudflare-ascii-pubkey.gpg
EOF

sudo dnf makecache
sudo dnf install -y cloudflared
cloudflared --version
```

### 1.2 Save Tunnel Connector Token & Create Systemd Service
1. In the **Cloudflare Zero Trust Dashboard (`one.dash.cloudflare.com`)** $\rightarrow$ **Networks** $\rightarrow$ **Tunnels** $\rightarrow$ **Create a tunnel** (`Cloudflared` connector) $\rightarrow$ name it `syntax-homelab` and copy your tunnel token (`eyJhIjoi...`).
2. Save the token securely on `syntax` in `/etc/cloudflared/token`:
   ```bash
   sudo install -d -m 700 /etc/cloudflared
   sudo install -m 600 /dev/null /etc/cloudflared/token
   # Paste ONLY the token string into /etc/cloudflared/token
   sudoedit /etc/cloudflared/token
   ```
3. Create `/etc/systemd/system/cloudflared.service`:
   ```bash
   sudo tee /etc/systemd/system/cloudflared.service > /dev/null << 'EOF'
   [Unit]
   Description=Cloudflare Tunnel client
   After=network-online.target
   Wants=network-online.target

   [Service]
   TimeoutStartSec=15
   Type=notify
   ExecStart=/usr/bin/cloudflared --no-autoupdate tunnel run --token-file /etc/cloudflared/token
   Restart=on-failure
   RestartSec=5s

   [Install]
   WantedBy=multi-user.target
   EOF

   sudo systemctl daemon-reload
   sudo systemctl enable --now cloudflared.service
   systemctl status cloudflared.service --no-pager
   ```

---

## 2. Cloudflare Zero Trust Dashboard Setup (`one.dash.cloudflare.com`)

### 2.1 Configure Published Application Routes
Under **Networks** $\rightarrow$ **Tunnels** $\rightarrow$ select your tunnel $\rightarrow$ **Public Hostname** tab, add two routes:

| Subdomain | Domain | Path | Service Type & URL | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `ssh` | `coinflow.lk` | `*` | `SSH` $\rightarrow$ `ssh://localhost:22` | Zero-Trust SSH terminal tunnel |
| `*` | `coinflow.lk` | `*` | `HTTP` $\rightarrow$ `http://localhost:80` | Wildcard web ingress to Nginx Proxy Manager |

> **⚠️ Important DNS Step for Wildcard (`*.coinflow.lk`):**
> Cloudflare's tunnel UI does not always auto-create wildcard DNS records. Go to **Cloudflare Dashboard -> Websites -> `coinflow.lk` -> DNS -> Records** and verify there is a **`CNAME`** record with:
> - **Name:** `*`
> - **Target:** `<YOUR_TUNNEL_ID_PLACEHOLDER>.cfargotunnel.com`
> - **Proxy status:** **Proxied (Orange Cloud ON)**

### 2.2 Configure Access Policies & Jellyfin Mobile App Bypass
Under **Access** $\rightarrow$ **Applications** $\rightarrow$ **Add an application** $\rightarrow$ **Self-hosted**:

1. **Protect Admin & Internal Apps (Optional OTP/Identity Policy):**
   - Add applications for sensitive subdomains (`npm.coinflow.lk`, `adguard.coinflow.lk`, `portainer.coinflow.lk`, `cockpit.coinflow.lk`, `torrent.coinflow.lk`, `radarr.coinflow.lk`, `sonarr.coinflow.lk`, `prowlarr.coinflow.lk`) with an **Allow** policy restricted to your email address.
2. **Jellyfin Mobile App Bypass (`tv.coinflow.lk`):**
   - Native Android/iOS Jellyfin apps cannot complete a browser-based Cloudflare Access login screen. Create a dedicated self-hosted application:
     - **Application Name:** `Jellyfin Bypass`
     - **Application Domain:** `tv.coinflow.lk`
     - **Policy Name:** `Bypass Everyone`
     - **Action:** **`Bypass`**
     - **Include Rule:** **`Everyone`**
3. **Service Token for Automated CLI / Hermes Agent (`ollama.coinflow.lk`):**
   - Go to **Access** $\rightarrow$ **Service Auth** $\rightarrow$ **Service Tokens** $\rightarrow$ **Create Service Token**.
   - Name it `Hermes API CLI Token`, click **Generate token**, and save the `Client ID` (`<PLACEHOLDER>`) and `Client Secret` (`<PLACEHOLDER>`).
   - Go to **Access** $\rightarrow$ **Applications** $\rightarrow$ select/create `ollama.coinflow.lk` $\rightarrow$ add a policy with **Action: `Service Auth`** and **Include: Service Token (`Hermes API CLI Token`)**.

---

## 3. Client Workstation Setup: How to SSH & Tunnel into the Home Server

To connect to `syntax` via `ssh.coinflow.lk` from your personal laptop/desktop (WSL2, Linux, Windows, or macOS), **the client machine also needs `cloudflared` installed** so OpenSSH can use `cloudflared access ssh` as its proxy command.

### 3.1 Install `cloudflared` on Your Client Machine
- **Linux / WSL2 (Installed to `~/.local/bin/cloudflared`):**
  ```bash
  mkdir -p ~/.local/bin
  curl -fsSL https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o ~/.local/bin/cloudflared
  chmod +x ~/.local/bin/cloudflared
  ~/.local/bin/cloudflared --version
  ```
- **Windows (PowerShell):**
  ```powershell
  winget install --id Cloudflare.cloudflared
  ```
- **macOS (Homebrew):**
  ```bash
  brew install cloudflared
  ```

### 3.2 Configure Client `~/.ssh/config` (`ssh syntax`)
Add the following profile to `~/.ssh/config` on your client workstation (and set `chmod 600 ~/.ssh/config`):

```sshconfig
Host syntax
    HostName ssh.coinflow.lk
    User sandun
    ProxyCommand ~/.local/bin/cloudflared access ssh --hostname %h
    LocalForward 3005 127.0.0.1:3005
    LocalForward 8081 127.0.0.1:81
    LocalForward 8082 192.168.1.1:80
```

### 3.3 Copy Your SSH Public Key for Passwordless Login
```bash
# Generate an Ed25519 SSH key on your client if you don't have one
ssh-keygen -t ed25519

# Copy your public key to the server
ssh-copy-id syntax
```

### 3.4 How to Use `ssh syntax` & Local Port Forwards
1. **Connect from Anywhere (Home or Outside):**
   ```bash
   ssh syntax
   ```
   While this SSH session stays open, your workstation automatically tunnels three internal admin UIs to `localhost`:
   - **`http://localhost:3005`** $\rightarrow$ AdGuard Home UI (`127.0.0.1:3005`)
   - **`http://localhost:8081`** $\rightarrow$ Nginx Proxy Manager Admin UI (`127.0.0.1:81`)
   - **`http://localhost:8082`** $\rightarrow$ ZTE Home Router Admin UI (`192.168.1.1:80`)
2. **Opening a Second Terminal Without Port Conflicts:**
   If one `ssh syntax` window is already holding ports `3005`/`8081`/`8082`, pass `-o ClearAllForwardings=yes` on extra terminals:
   ```bash
   ssh -o ClearAllForwardings=yes syntax
   ```
3. **Direct LAN SSH Fallback (When on Home Wi-Fi):**
   ```bash
   ssh sandun@192.168.1.3
   ```

---

## 4. Host Prerequisites for Edge Stack (Port 53 & Cockpit Proxy)

### 4.1 Free Host Port 53 (`systemd-resolved` Stub Listener Fix)
Before deploying AdGuard Home (`npm-stack.yml`), ensure `systemd-resolved` is not occupying port 53:
```bash
sudo ss -tulpn | grep :53

sudo mkdir -p /etc/systemd/resolved.conf.d
sudo tee /etc/systemd/resolved.conf.d/adguardhome.conf > /dev/null << 'EOF'
[Resolve]
DNS=127.0.0.1
DNSStubListener=no
EOF

sudo systemctl reload-or-restart systemd-resolved
sudo ss -tulpn | grep :53
```

### 4.2 Configure RHEL Cockpit for Reverse Proxy (`cockpit.coinflow.lk`)
Without this config, Cockpit rejects proxied WebSocket connections or enters a login loop:
```bash
sudo tee /etc/cockpit/cockpit.conf > /dev/null << 'EOF'
[WebService]
Origins = https://cockpit.coinflow.lk wss://cockpit.coinflow.lk https://192.168.1.3:9090
ProtocolHeader = X-Forwarded-Proto
ForwardedForHeader = X-Forwarded-For
AllowUnencrypted = true
UrlRoot = /
EOF

sudo systemctl enable --now cockpit.socket
sudo systemctl restart cockpit
```

### 4.3 Deploy Edge Stack (`npm-stack.yml` & `portainer-stack.yml`)
```bash
docker stack deploy -c ~/portainer-stack.yml portainer
docker stack deploy -c ~/npm-stack.yml edge
docker service ls
```

---

## 5. Nginx Proxy Manager (NPM) Setup (`http://192.168.1.3:81`)

### 5.1 Request a Let's Encrypt Wildcard SSL Certificate
1. Open NPM Admin at `http://192.168.1.3:81` (or `http://localhost:8081` over SSH).
2. Go to **SSL Certificates** $\rightarrow$ **Add SSL Certificate** $\rightarrow$ **Let's Encrypt**:
   - **Domain Names:** `coinflow.lk`, `*.coinflow.lk`
   - **Use a DNS Challenge:** **ON**
   - **DNS Provider:** `Cloudflare`
   - **Credentials File Content:** `dns_cloudflare_api_token = <CLOUDFLARE_DNS_API_TOKEN_PLACEHOLDER>`
   - Click **Save**.

### 5.2 Create All 14 Proxy Hosts
Assign the wildcard SSL certificate (with **HTTP/2 Support ON**) to all 14 hosts:

| Domain Name | Scheme | Forward Hostname / IP | Forward Port | WebSockets | Block Exploits | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `npm.coinflow.lk` | `http` | `edge_npm` | `81` | ON | OFF | NPM Admin UI |
| `adguard.coinflow.lk` | `http` | `edge_adguard` | `3000` | ON | **OFF** | Keep exploits OFF for DNS/API |
| `ai.coinflow.lk` | `http` | `ai_stack_open-webui` | `8080` | ON | ON | WebSockets required for LLM chat |
| `cockpit.coinflow.lk` | `https` | `192.168.1.3` | `9090` | **ON** | **OFF** | Routes to host `:9090` |
| `portainer.coinflow.lk` | `https` | `portainer_portainer` | `9443` | ON | ON | Via `portainer_portainer_net` |
| `home.coinflow.lk` | `http` | `homepage_homepage` | `3000` | OFF | ON | Homepage dashboard |
| `ollama.coinflow.lk` | `http` | `ai_stack_ollama` | `11434` | ON | ON | Raw Ollama API |
| `request.coinflow.lk` | `http` | `media_jellyseerr` | `5055` | ON | ON | Seerr request portal |
| `tv.coinflow.lk` | `http` | `media_jellyfin` | `8096` | **ON** | OFF | Force SSL **OFF** + Advanced block below |
| `prowlarr.coinflow.lk` | `http` | `media_prowlarr` | `9696` | OFF | ON | Indexer manager |
| `radarr.coinflow.lk` | `http` | `media_radarr` | `7878` | OFF | ON | Movie manager |
| `sonarr.coinflow.lk` | `http` | `media_sonarr` | `8989` | OFF | ON | TV series manager |
| `torrent.coinflow.lk` | `http` | `media_qbittorrent` | `8080` | OFF | ON | qBittorrent WebUI |
| `xx.coinflow.lk` | `http` | `192.168.1.3` | `9999` | **ON** | ON | Stash + LAN-only lock below |

#### Advanced Tab Rule 1: `tv.coinflow.lk` (Allow Remote Browsing, Block Internet Video Playback)
```nginx
proxy_buffering off;

# Block video playback if traffic comes through Cloudflare Tunnel (Internet)
# Local Wi-Fi traffic goes directly to 192.168.1.3 (no http_cf_connecting_ip header) and plays normally!
location ~* ^/(Videos|Audio|LiveStreams)/ {
    if ($http_cf_connecting_ip != "") {
        return 403 "Video streaming is disabled over the internet. Connect to Home Wi-Fi to watch.";
    }

    proxy_pass http://192.168.1.3:8096;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Protocol $scheme;
    proxy_set_header X-Forwarded-Host $http_host;
    proxy_buffering off;
}
```

#### Advanced Tab Rule 2: `xx.coinflow.lk` (Strict LAN-Only Access)
```nginx
proxy_buffering off;

# Block ALL outside internet access coming through Cloudflare Proxy/Tunnel
if ($http_cf_connecting_ip != "") {
    return 403 "Access denied. Connect to Home LAN/Wi-Fi.";
}
```

---

## 6. AdGuard Home Split-DNS & ZTE Router Configuration

### 6.1 Initial AdGuard Home Setup (`http://192.168.1.3:3005`)
1. On first launch, visit `http://192.168.1.3:3005` (published to container port `3000` during setup; once configured, the admin UI stays accessible on `:3005` and `https://adguard.coinflow.lk`).
2. **Filters -> DNS rewrites:**
   - Add Domain: `*.coinflow.lk` $\rightarrow$ Answer: `192.168.1.3`
3. **Filters -> Custom filtering rules (IPv6 `AAAA` Leak Prevention):**
   - Add the rule below so phones and Windows PCs on home Wi-Fi do not resolve `*.coinflow.lk` to Cloudflare's public IPv6 (`AAAA`) addresses and bypass local LAN routing:
     ```text
     ||coinflow.lk^$dnstype=AAAA
     ```
4. **Settings -> DNS settings -> Upstream DNS servers:**
   ```text
   https://dns.cloudflare.com/dns-query
   https://dns.quad9.net/dns-query
   tls://dns.adguard-dns.com
   ```
5. **Filters -> DNS blocklists:** Enable *AdGuard DNS filter*, *AdAway Default*, *HaGeZi Normal*, *OISD Big*, and *Dandelion Sprout Anti-Malware*.

### 6.2 ZTE F660 Home Router Setup (`http://192.168.1.1` or `http://localhost:8082`)
1. **LAN -> DHCP Binding:** Bind the `syntax` laptop MAC address to `192.168.1.3`.
2. **LAN -> DHCP Server:**
   - **Primary DNS:** `192.168.1.3` (AdGuard Home)
   - **Secondary / Tertiary DNS:** `192.168.1.1` (Router fallback if server is rebooting — **never** put `8.8.8.8` or `1.1.1.1` here or client devices will randomly bypass Split-DNS!).
3. **Application -> DNS Service -> Hosts:** Add `tv.coinflow.lk` $\rightarrow$ `192.168.1.3`.
4. **Disable LAN IPv6 (DHCPv6 & RA):** Turn off IPv6 Router Advertisements on LAN so mobile devices strictly use IPv4 AdGuard DNS.

### 6.3 Test Split-DNS Resolution
```bash
# Should return 192.168.1.3 (IPv4) and 0 AAAA records
nslookup tv.coinflow.lk 192.168.1.3
nslookup -type=AAAA tv.coinflow.lk 192.168.1.3
```

---

➡️ **Next Step:** Proceed to **[04-Media-and-Servarr-Setup.md](./04-Media-and-Servarr-Setup.md)** to deploy and connect the Servarr + Jellyfin + Stash media stack.
