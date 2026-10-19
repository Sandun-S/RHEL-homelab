# 🛠️ Complete RHEL 10.1 Home Lab Build, Operations & Access Guide

This comprehensive guide documents every step required to **access**, **build**, **configure**, and **maintain** the **`syntax`** (`192.168.1.3` / `coinflow.lk`) RHEL 10.1 Home Server from bare metal to full application interconnections.

> **Security Note:** All API keys, tokens, and passwords in this guide are replaced with `<PLACEHOLDER>` values.

---

## 📑 Table of Contents

1. **[Part 1: Connecting to the Working Home Server (Client & Remote Access)](#part-1-connecting-to-the-working-home-server-client--remote-access)**
   - [1.1 Web Access: At Home (Split-DNS) vs. Remote (Cloudflare Tunnel)](#11-web-access-at-home-split-dns-vs-remote-cloudflare-tunnel)
   - [1.2 Client Workstation Setup: Installing `cloudflared` Locally](#12-client-workstation-setup-installing-cloudflared-locally)
   - [1.3 SSH Proxy Configuration & Local Port Forwarding (`ssh syntax`)](#13-ssh-proxy-configuration--local-port-forwarding-ssh-syntax)
2. **[Part 2: Base OS Installation, UEFI Fix & Hardware Hardening](#part-2-base-os-installation-uefi-fix--hardware-hardening)**
   - [2.1 RHEL 10.1 Anaconda Offline Install on External USB NVMe](#21-rhel-101-anaconda-offline-install-on-external-usb-nvme)
   - [2.2 UEFI Removable Media Bootloader Fix (`BOOTX64.EFI`)](#22-uefi-removable-media-bootloader-fix-bootx64efi)
   - [2.3 24/7 Lid-Closed Screen Blanking, SSD Trim & KVM Virtualization](#23-247-lid-closed-screen-blanking-ssd-trim--kvm-virtualization)
   - [2.4 Docker CE Logging, Firewalld & Swarm Initialization](#24-docker-ce-logging-firewalld--swarm-initialization)
3. **[Part 3: Hybrid 3-Tier Storage Engineering (LVM, NTFS, BitLocker & SMB)](#part-3-hybrid-3-tier-storage-engineering-lvm-ntfs-bitlocker--smb)**
   - [3.1 Mounting Secondary NTFS Partition (`/dev/sda2` -> `/mnt/media_drive`)](#31-mounting-secondary-ntfs-partition-devsda2---mntmedia_drive)
   - [3.2 Online LVM `/home` Reclamation into `/` (62 GB XFS Root)](#32-online-lvm-home-reclamation-into--62-gb-xfs-root)
   - [3.3 Automated Boot-Time BitLocker Unlock (`/dev/sdb1` -> `/mnt/win_1tb`)](#33-automated-boot-time-bitlocker-unlock-devsdb1---mntwin_1tb)
   - [3.4 Windows Desktop SMB Shares & Fault-Tolerant `x-systemd.automount`](#34-windows-desktop-smb-shares--fault-tolerant-x-systemdautomount)
   - [3.5 Safe Unmount / Remount Procedure for `/mnt/win_1tb`](#35-safe-unmount--remount-procedure-for-mntwin_1tb)
4. **[Part 4: Cloudflare Zero Trust, Nginx Proxy Manager & AdGuard Split-DNS](#part-4-cloudflare-zero-trust-nginx-proxy-manager--adguard-split-dns)**
   - [4.1 Host `cloudflared.service` Installation & Systemd Unit](#41-host-cloudflaredservice-installation--systemd-unit)
   - [4.2 Cloudflare Zero Trust Dashboard Configuration (Tunnels, Bypass & Service Tokens)](#42-cloudflare-zero-trust-dashboard-configuration-tunnels-bypass--service-tokens)
   - [4.3 Resolving Port 53 Conflict (`systemd-resolved`) & Cockpit Proxy Headers](#43-resolving-port-53-conflict-systemd-resolved--cockpit-proxy-headers)
   - [4.4 Nginx Proxy Manager (NPM): All 14 Host Rules & Custom Security Locations](#44-nginx-proxy-manager-npm-all-14-host-rules--custom-security-locations)
   - [4.5 AdGuard Home Split-DNS, `AAAA` Leak Prevention & ZTE Router Setup](#45-adguard-home-split-dns-aaaa-leak-prevention--zte-router-setup)
5. **[Part 5: Master Docker Swarm Stack Deployment & Homepage Dashboard](#part-5-master-docker-swarm-stack-deployment--homepage-dashboard)**
   - [5.1 Creating Host Prerequisite Directories](#51-creating-host-prerequisite-directories)
   - [5.2 Deploying All 5 Docker Swarm Stacks](#52-deploying-all-5-docker-swarm-stacks)
   - [5.3 Homepage Dashboard (`Syntax Edge Hub`) Configuration](#53-homepage-dashboard-syntax-edge-hub-configuration)
6. **[Part 6: Servarr & Media Stack Interconnections (qBit, Prowlarr, Radarr, Sonarr, Seerr, Jellyfin, Stash)](#part-6-servarr--media-stack-interconnections-qbit-prowlarr-radarr-sonarr-seerr-jellyfin-stash)**
   - [6.1 Internal Swarm Service Addresses & Root Folder Mappings](#61-internal-swarm-service-addresses--root-folder-mappings)
   - [6.2 Step-by-Step App Wiring (qBittorrent -> Prowlarr -> Radarr/Sonarr -> Seerr -> Jellyfin)](#62-step-by-step-app-wiring-qbittorrent---prowlarr---radarrsonarr---seerr---jellyfin)
   - [6.3 Stash Setup & Security Tripwire Reset](#63-stash-setup--security-tripwire-reset)
   - [6.4 Media Library Maintenance & Automated `.zip` Unpack Script](#64-media-library-maintenance--automated-zip-unpack-script)
7. **[Part 7: Edge AI Stack, Model Downloads, Hermes Agent & `ask` CLI](#part-7-edge-ai-stack-model-downloads-hermes-agent--ask-cli)**
   - [7.1 Pulling Ollama Models & Off-Peak Nightly Scheduled Downloads](#71-pulling-ollama-models--off-peak-nightly-scheduled-downloads)
   - [7.2 NousResearch Hermes Agent Installation & Cloudflare Service Token Auth](#72-nousresearch-hermes-agent-installation--cloudflare-service-token-auth)
   - [7.3 Terminal CLI Assistant (`ask` + `glow`) Setup on RHEL 10.1](#73-terminal-cli-assistant-ask--glow-setup-on-rhel-101)

---

## Part 1: Connecting to the Working Home Server (Client & Remote Access)

### 1.1 Web Access: At Home (Split-DNS) vs. Remote (Cloudflare Tunnel)

You use the **exact same `https://*.coinflow.lk` URLs** both at home and away:

- **🏠 At Home (Home Wi-Fi / LAN):**
  - Your device receives `192.168.1.3` (AdGuard Home) as its DNS server via router DHCP.
  - AdGuard rewrites `*.coinflow.lk` directly to `192.168.1.3` and blocks IPv6 (`AAAA`) responses (`||coinflow.lk^$dnstype=AAAA`).
  - Traffic routes directly over **local Gigabit LAN** to Nginx Proxy Manager (no Cloudflare hop). Full video streaming on `https://tv.coinflow.lk` and access to `https://xx.coinflow.lk` work unrestricted.
- **🌍 Away from Home (Mobile Data / External Network):**
  - Public DNS routes `*.coinflow.lk` through the **Cloudflare Zero Trust Tunnel** (`cloudflared.service`).
  - Protected dashboards require Cloudflare Access authentication. `https://tv.coinflow.lk` permits remote catalog browsing and Jellyseerr requests (`https://request.coinflow.lk`), while blocking high-bandwidth video playback over the internet to comply with Cloudflare ToS.

| Service | Domain URL | Direct LAN Fallback |
| :--- | :--- | :--- |
| **Homepage Dashboard** | `https://home.coinflow.lk` | `http://192.168.1.3:3001` |
| **Open WebUI (AI Chat)** | `https://ai.coinflow.lk` | `http://192.168.1.3:3000` |
| **Ollama Inference API** | `https://ollama.coinflow.lk` | `http://192.168.1.3:11434` |
| **Jellyfin Media Server** | `https://tv.coinflow.lk` | `http://192.168.1.3:8096` |
| **Jellyseerr (Seerr Requests)** | `https://request.coinflow.lk` | *(Routed via `hub_net`)* |
| **qBittorrent WebUI** | `https://torrent.coinflow.lk` | *(Routed via `hub_net`)* |
| **Radarr (Movies)** | `https://radarr.coinflow.lk` | *(Routed via `hub_net`)* |
| **Sonarr (TV Shows)** | `https://sonarr.coinflow.lk` | *(Routed via `hub_net`)* |
| **Prowlarr (Indexers)** | `https://prowlarr.coinflow.lk` | *(Routed via `hub_net`)* |
| **Stash** | `https://xx.coinflow.lk` | `http://192.168.1.3:9999` *(LAN only)* |
| **Portainer CE** | `https://portainer.coinflow.lk` | `https://192.168.1.3:9443` |
| **Nginx Proxy Manager** | `https://npm.coinflow.lk` | `http://192.168.1.3:81` |
| **AdGuard Home** | `https://adguard.coinflow.lk` | `http://192.168.1.3:3005` |
| **RHEL Cockpit + KVM** | `https://cockpit.coinflow.lk` | `https://192.168.1.3:9090` |

---

### 1.2 Client Workstation Setup: Installing `cloudflared` Locally

To SSH into the home server via `ssh.coinflow.lk` from your laptop/desktop (Linux, WSL2, macOS, or Windows), you must install the `cloudflared` binary on the **client machine** as well:

- **Linux / WSL2 (User-local binary in `~/.local/bin/cloudflared`):**
  ```bash
  mkdir -p ~/.local/bin
  curl -fsSL https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o ~/.local/bin/cloudflared
  chmod +x ~/.local/bin/cloudflared
  ~/.local/bin/cloudflared --version
  ```
- **Windows (PowerShell / Winget):**
  ```powershell
  winget install --id Cloudflare.cloudflared
  ```
- **macOS (Homebrew):**
  ```bash
  brew install cloudflared
  ```

---

### 1.3 SSH Proxy Configuration & Local Port Forwarding (`ssh syntax`)

Add the following block to `~/.ssh/config` on your client workstation:

```sshconfig
Host syntax
    HostName ssh.coinflow.lk
    User sandun
    ProxyCommand ~/.local/bin/cloudflared access ssh --hostname %h
    LocalForward 3005 127.0.0.1:3005
    LocalForward 8081 127.0.0.1:81
    LocalForward 8082 192.168.1.1:80
```

#### How to Connect:
1. **Via Zero-Trust SSH Tunnel (Works from Anywhere + Opens Local Port Forwards):**
   ```bash
   ssh syntax
   ```
   While this session is open, you can open these URLs in your local workstation browser:
   - **`http://localhost:3005`** $\rightarrow$ AdGuard Home UI on `syntax` (`127.0.0.1:3005`)
   - **`http://localhost:8081`** $\rightarrow$ Nginx Proxy Manager Admin UI on `syntax` (`127.0.0.1:81`)
   - **`http://localhost:8082`** $\rightarrow$ ZTE F660 Home Router UI (`192.168.1.1:80`)

2. **Running Additional SSH Commands When a Session is Already Open:**
   To avoid `bind: Address already in use` on ports `3005`/`8081`/`8082` when opening a second terminal or running scripts:
   ```bash
   ssh -o ClearAllForwardings=yes syntax
   ```

3. **Direct LAN SSH (When on Home Wi-Fi without Cloudflare):**
   ```bash
   ssh sandun@192.168.1.3
   ```

---

## Part 2: Base OS Installation, UEFI Fix & Hardware Hardening

### 2.1 RHEL 10.1 Anaconda Offline Install on External USB NVMe
1. Flash the **RHEL 10.1 Boot ISO** to a USB stick using Rufus or BalenaEtcher.
2. In the Anaconda installer:
   - Skip network setup during graphical installation (offline minimal install).
   - Select the external USB NVMe (`/dev/sda`) as the installation target, preserving `/dev/sda2` (50 GB NTFS backup partition) and leaving the internal Windows disks (`/dev/nvme0n1` and `/dev/sdb`) completely untouched.
   - *Tip:* Use `Ctrl + Alt + F2` for root TTY console during install, and `Ctrl + Alt + F6` to return to the GUI.

### 2.2 UEFI Removable Media Bootloader Fix (`BOOTX64.EFI`)
Standard PC UEFI firmware looks for removable bootloaders at `EFI/BOOT/BOOTX64.EFI`, whereas RHEL installs to `EFI/redhat/`. Copy the EFI binaries and automate future kernel updates:

```bash
sudo mkdir -p /boot/efi/EFI/BOOT/
sudo cp -rf /boot/efi/EFI/redhat/* /boot/efi/EFI/BOOT/
sudo cp -f /boot/efi/EFI/redhat/shimx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI

cat << 'EOF' | sudo tee /etc/kernel/postinst.d/99-sync-usb-efi
#!/bin/sh
cp -rf /boot/efi/EFI/redhat/* /boot/efi/EFI/BOOT/ 2>/dev/null
cp -f /boot/efi/EFI/redhat/shimx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI 2>/dev/null
EOF
sudo chmod +x /etc/kernel/postinst.d/99-sync-usb-efi
```

### 2.3 24/7 Lid-Closed Screen Blanking, SSD Trim & KVM Virtualization

```bash
# 1. Blank console immediately or wake it manually
sudo setterm --term linux --blank force </dev/tty1 >/dev/tty1
sudo setterm --term linux --blank poke </dev/tty1 >/dev/tty1

# 2. Persist 60-second console blanking across reboots via kernel cmdline
sudo grubby --update-kernel=ALL --args="consoleblank=60"

# 3. Enable weekly SSD TRIM for the external NVMe
sudo systemctl enable --now fstrim.timer

# 4. Install KVM / Libvirt virtualization for Cockpit Machines
sudo dnf install -y qemu-kvm libvirt virt-install cockpit-machines
sudo systemctl enable --now libvirtd
```

### 2.4 Docker CE Logging, Firewalld & Swarm Initialization

1. **Configure Docker Log Rotation (`/etc/docker/daemon.json`):**
   ```json
   {
     "log-driver": "json-file",
     "log-opts": {
       "max-size": "10m",
       "max-file": "3"
     }
   }
   ```
2. **Open Firewalld Ports & Initialize Docker Swarm:**
   ```bash
   sudo firewall-cmd --permanent --add-port={2377/tcp,7946/tcp,7946/udp,4789/udp}
   sudo firewall-cmd --permanent --add-service={dns,cockpit}
   sudo firewall-cmd --permanent --zone=trusted --add-interface=docker_gwbridge
   sudo firewall-cmd --reload

   docker swarm init
   docker network create --driver overlay --attachable hub_net
   ```

---

## Part 3: Hybrid 3-Tier Storage Engineering (LVM, NTFS, BitLocker & SMB)

### 3.1 Mounting Secondary NTFS Partition (`/dev/sda2` -> `/mnt/media_drive`)
```bash
sudo dnf install -y epel-release
sudo dnf install -y ntfs-3g
sudo mkdir -p /mnt/media_drive
sudo mount -t ntfs-3g -o uid=1000,gid=1000,umask=022 /dev/sda2 /mnt/media_drive
echo "UUID=2C66F38766F34FD6 /mnt/media_drive ntfs-3g uid=1000,gid=1000,umask=022 0 0" | sudo tee -a /etc/fstab
```

### 3.2 Online LVM `/home` Reclamation into `/` (62 GB XFS Root)
Because Docker images in `/var/lib/docker` consume root space and XFS cannot be shrunk online, the 20.3 GB `rhel_syntax-home` LV was backed up, removed, and merged into `rhel_syntax-root`:

```bash
sudo cp -a /home/sandun /mnt/media_drive/sandun_backup
sudo -i
systemctl stop docker docker.socket
pkill -u sandun
fuser -vkm -9 /home
umount /home
lvremove -y /dev/rhel_syntax/home
lvextend -r -l +100%FREE /dev/rhel_syntax/root
sed -i '\|/home|d' /etc/fstab
mkdir -p /home
cp -a /mnt/media_drive/sandun_backup /home/sandun
chown -R sandun:sandun /home/sandun
restorecon -R /home/sandun
systemctl start docker
```

### 3.3 Automated Boot-Time BitLocker Unlock (`/dev/sdb1` -> `/mnt/win_1tb`)
Unlock the internal 1 TB Windows BitLocker HDD (`PARTUUID=7937ddea-af22-4c7e-8660-2510540ef0a7`) automatically at boot using `systemd-cryptsetup`:

```bash
# 1. Create protected key file (root:root 0600) and paste the 48-digit BitLocker recovery key
sudo install -d -m 700 /etc/cryptsetup-keys.d
sudo install -m 600 /dev/null /etc/cryptsetup-keys.d/win_1tb.key
sudoedit /etc/cryptsetup-keys.d/win_1tb.key

# 2. Add BitLocker mapping to /etc/crypttab
echo "win_1tb PARTUUID=7937ddea-af22-4c7e-8660-2510540ef0a7 /etc/cryptsetup-keys.d/win_1tb.key bitlk" | sudo tee -a /etc/crypttab

# 3. Add NTFS mount to /etc/fstab
echo "/dev/mapper/win_1tb /mnt/win_1tb ntfs-3g rw,uid=1000,gid=1000,umask=022 0 0" | sudo tee -a /etc/fstab

# 4. Reload systemd, unlock, and mount
sudo mkdir -p /mnt/win_1tb
sudo systemctl daemon-reload
sudo systemctl start systemd-cryptsetup@win_1tb.service
sudo udevadm settle
sudo mount /mnt/win_1tb
```

### 3.4 Windows Desktop SMB Shares & Fault-Tolerant `x-systemd.automount`
1. **On Windows Desktop (`192.168.1.2`):**
   - Share folders `Movies`, `TV_Shows`, and `001` with `Everyone` (**Read/Change** permissions).
   - In **Network & Sharing Center -> All Networks**, turn **OFF** Password Protected Sharing.
2. **On RHEL Server (`192.168.1.3`):**
   ```bash
   sudo dnf install -y cifs-utils samba-client
   sudo mkdir -p /mnt/desktop_movies /mnt/desktop_tv_shows /mnt/desktop_001
   ```
   Add fault-tolerant automount lines to `/etc/fstab` (mounts on demand, unmounts after 60s idle, and skips in 5s if the desktop PC is powered off):
   ```fstab
   //192.168.1.2/Movies    /mnt/desktop_movies    cifs  username=guest,password=,uid=1000,gid=1000,vers=3.0,soft,_netdev,nofail,x-systemd.automount,x-systemd.idle-timeout=60,x-systemd.mount-timeout=5s  0  0
   //192.168.1.2/TV_Shows  /mnt/desktop_tv_shows  cifs  username=guest,password=,uid=1000,gid=1000,vers=3.0,soft,_netdev,nofail,x-systemd.automount,x-systemd.idle-timeout=60,x-systemd.mount-timeout=5s  0  0
   //192.168.1.2/001       /mnt/desktop_001       cifs  username=guest,password=,uid=1000,gid=1000,vers=3.0,soft,_netdev,nofail,x-systemd.automount,x-systemd.idle-timeout=60,x-systemd.mount-timeout=5s  0  0
   ```
   Apply and test:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart remote-fs.target
   ls -la /mnt/desktop_movies /mnt/desktop_tv_shows /mnt/desktop_001
   ```

### 3.5 Safe Unmount / Remount Procedure for `/mnt/win_1tb`
Before unmounting `/mnt/win_1tb`, scale down the Swarm services holding active bind mounts:
```bash
docker service scale media_jellyfin=0 media_qbittorrent=0 media_radarr=0 media_sonarr=0 media_stash=0
sudo umount /mnt/win_1tb
sudo systemctl stop systemd-cryptsetup@win_1tb.service
sudo cryptsetup close win_1tb
sudo udevadm settle

# Re-unlock, mount, and scale back up
sudo systemctl start systemd-cryptsetup@win_1tb.service
sudo udevadm settle
sudo mount /mnt/win_1tb
docker service scale media_jellyfin=1 media_qbittorrent=1 media_radarr=1 media_sonarr=1 media_stash=1
```

---

## Part 4: Cloudflare Zero Trust, Nginx Proxy Manager & AdGuard Split-DNS

### 4.1 Host `cloudflared.service` Installation & Systemd Unit

1. **Install `cloudflared` on RHEL 10.1 Host (`syntax`):**
   ```bash
   sudo curl -fsSL https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o /usr/bin/cloudflared
   sudo chmod +x /usr/bin/cloudflared
   ```
2. **Store Tunnel Connector Token:**
   ```bash
   sudo install -d -m 700 /etc/cloudflared
   sudo install -m 600 /dev/null /etc/cloudflared/token
   # Paste your Cloudflare Tunnel token into /etc/cloudflared/token
   sudoedit /etc/cloudflared/token
   ```
3. **Create `/etc/systemd/system/cloudflared.service`:**
   ```ini
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
   ```
   Enable and start:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable --now cloudflared.service
   ```

### 4.2 Cloudflare Zero Trust Dashboard Configuration (Tunnels, Bypass & Service Tokens)

In the Cloudflare Zero Trust Dashboard (`one.dash.cloudflare.com`):

1. **Published Tunnel Routes (`Networks -> Tunnels -> Public Hostnames`):**
   - `ssh.coinflow.lk` $\rightarrow$ `ssh://localhost:22`
   - `*.coinflow.lk` $\rightarrow$ `http://localhost:80`
   - *(In Cloudflare DNS, create a proxied Orange-Cloud wildcard `CNAME` for `*` pointing to `<CLOUDFLARE_TUNNEL_ID>.cfargotunnel.com`.)*
2. **Jellyfin Mobile App Bypass (`Access -> Applications -> Add an application -> Self-hosted`):**
   - **Application Name:** `Jellyfin Bypass`
   - **Domain:** `tv.coinflow.lk`
   - **Policy Action:** `Bypass` | **Include Rule:** `Everyone` *(Allows native Android/iOS Jellyfin clients to connect without an HTML login prompt).*
3. **Hermes Agent CLI Service Token (`Access -> Service Auth -> Service Tokens`):**
   - Create token **`Hermes API CLI Token`** and copy the `Client ID` (`<PLACEHOLDER>`) and `Client Secret` (`<PLACEHOLDER>`).
   - Under **Access -> Applications**, create/edit the application for `ollama.coinflow.lk` and add a policy with **Action: `Service Auth`** and **Include: Service Token (`Hermes API CLI Token`)**.

### 4.3 Resolving Port 53 Conflict (`systemd-resolved`) & Cockpit Proxy Headers

1. **Disable `systemd-resolved` Stub Listener on Port 53 (So AdGuard Can Bind `0.0.0.0:53`):**
   ```bash
   sudo mkdir -p /etc/systemd/resolved.conf.d
   cat << 'EOF' | sudo tee /etc/systemd/resolved.conf.d/adguardhome.conf
   [Resolve]
   DNS=127.0.0.1
   DNSStubListener=no
   EOF
   sudo systemctl reload-or-restart systemd-resolved
   sudo ss -tulpn | grep :53
   ```
2. **Configure RHEL Cockpit for Reverse Proxying (`/etc/cockpit/cockpit.conf`):**
   ```bash
   sudo tee /etc/cockpit/cockpit.conf << 'EOF'
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

### 4.4 Nginx Proxy Manager (NPM): All 14 Host Rules & Custom Security Locations

Access NPM Admin UI at `http://192.168.1.3:81` (or `http://localhost:8081` via SSH forward) and configure the following Proxy Hosts (all sharing a wildcard/Let's Encrypt SSL cert with HTTP/2 enabled):

| Domain Name | Scheme | Forward Hostname / IP | Port | WebSockets | Block Exploits | Special Config |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `npm.coinflow.lk` | `http` | `edge_npm` | `81` | ON | OFF | NPM Admin UI |
| `adguard.coinflow.lk` | `http` | `edge_adguard` | `3000` | ON | **OFF** | Keep exploits OFF for DNS/API queries |
| `ai.coinflow.lk` | `http` | `ai_stack_open-webui` | `8080` | ON | ON | WebSockets required for streaming tokens |
| `cockpit.coinflow.lk` | `https` | `192.168.1.3` | `9090` | **ON** | **OFF** | Host IP `:9090` |
| `portainer.coinflow.lk` | `https` | `portainer_portainer` | `9443` | ON | ON | Routed via `portainer_portainer_net` |
| `home.coinflow.lk` | `http` | `homepage_homepage` | `3000` | OFF | ON | Homepage dashboard |
| `ollama.coinflow.lk` | `http` | `ai_stack_ollama` | `11434` | ON | ON | Raw Ollama API |
| `request.coinflow.lk` | `http` | `media_jellyseerr` | `5055` | ON | ON | Seerr request portal |
| `tv.coinflow.lk` | `http` | `media_jellyfin` | `8096` | **ON** | OFF | Force SSL **OFF** + Custom stream block below |
| `prowlarr.coinflow.lk` | `http` | `media_prowlarr` | `9696` | OFF | ON | Indexer manager |
| `radarr.coinflow.lk` | `http` | `media_radarr` | `7878` | OFF | ON | Movie manager |
| `sonarr.coinflow.lk` | `http` | `media_sonarr` | `8989` | OFF | ON | TV series manager |
| `torrent.coinflow.lk` | `http` | `media_qbittorrent` | `8080` | OFF | ON | qBittorrent WebUI |
| `xx.coinflow.lk` | `http` | `192.168.1.3` | `9999` | **ON** | ON | Stash + LAN-only lock below |

#### Custom NPM Advanced Config 1: Block Remote Internet Video Streaming on `tv.coinflow.lk`
Paste into **NPM -> Proxy Hosts -> `tv.coinflow.lk` -> Advanced Tab**:
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

#### Custom NPM Advanced Config 2: Strict LAN-Only Lock on `xx.coinflow.lk`
Paste into **NPM -> Proxy Hosts -> `xx.coinflow.lk` -> Advanced Tab**:
```nginx
proxy_buffering off;

# Block ALL outside internet access coming through Cloudflare Proxy/Tunnel
if ($http_cf_connecting_ip != "") {
    return 403 "Access denied. Connect to Home LAN/Wi-Fi.";
}
```

### 4.5 AdGuard Home Split-DNS, `AAAA` Leak Prevention & ZTE Router Setup

1. **AdGuard Home (`http://192.168.1.3:3005` or `https://adguard.coinflow.lk`):**
   - **Filters -> DNS rewrites:** Add wildcard `*.coinflow.lk` $\rightarrow$ `192.168.1.3`.
   - **Filters -> Custom filtering rules:** Block IPv6 `AAAA` queries for `coinflow.lk` so dual-stack devices never bypass local IPv4 Split-DNS:
     ```text
     ||coinflow.lk^$dnstype=AAAA
     ```
   - **Settings -> DNS settings -> Upstream DNS servers:**
     ```text
     https://dns.cloudflare.com/dns-query
     https://dns.quad9.net/dns-query
     tls://dns.adguard-dns.com
     ```
2. **ZTE F660 Home Router (`192.168.1.1`):**
   - **Static DHCP Binding:** Bind `syntax` laptop MAC `<LAPTOP_MAC_ADDRESS>` to `192.168.1.3`.
   - **LAN -> DHCP Server:** Set **DNS Server 1** = `192.168.1.3` (AdGuard Home) and **DNS Server 2 / 3** = `192.168.1.1` (Router fallback if server is offline; do not use `1.1.1.1` or `8.8.8.8`).
   - **IPv6 Disable:** Turn **OFF** DHCPv6 and Router Advertisements (RA) on LAN so phones/PCs use IPv4 AdGuard DNS exclusively.

---

## Part 5: Master Docker Swarm Stack Deployment & Homepage Dashboard

### 5.1 Creating Host Prerequisite Directories
Docker Swarm requires all bind-mounted directories to exist on the host before deploying stacks:
```bash
mkdir -p ~/homepage/config
mkdir -p ~/media-stack/config/{qbittorrent,prowlarr,radarr,sonarr,jellyseerr,jellyfin,stash/generated}
sudo mkdir -p /mnt/media_drive/data/{torrents/{movies,tv},media/{movies,tv}}
sudo mkdir -p /mnt/win_1tb/{Movies,TV_Shows,torrents}
sudo chown -R 1000:1000 /mnt/media_drive/data ~/media-stack/config
```

### 5.2 Deploying All 5 Docker Swarm Stacks
```bash
# Ensure the shared external overlay network exists
docker network create --driver overlay --attachable hub_net

# Deploy all 5 stacks
docker stack deploy -c ~/portainer-stack.yml portainer
docker stack deploy -c ~/npm-stack.yml edge
docker stack deploy -c ~/ai-iot-stack.yml ai_stack
docker stack deploy -c ~/homepage-stack.yml homepage
docker stack deploy -c ~/media-stack.yml media

# Verify service replicas
docker service ls
```

### 5.3 Homepage Dashboard (`Syntax Edge Hub`) Configuration
- Configuration files live in `~/homepage/config/` (`services.yaml`, `widgets.yaml`, `docker.yaml`, `settings.yaml`, `custom.css`, `bookmarks.yaml`).
- In `homepage-stack.yml`, `HOMEPAGE_ALLOWED_HOSTS=home.coinflow.lk,localhost:3001` prevents HTTP `403 Forbidden` host validation errors.
- If enabling the Cloudflare Tunnel status widget in `~/homepage/config/services.yaml`, populate `accountid`, `tunnelid`, and `key` with `<PLACEHOLDER>` or environment variables so API tokens are never committed to Git.

---

## Part 6: Servarr & Media Stack Interconnections (qBit, Prowlarr, Radarr, Sonarr, Seerr, Jellyfin, Stash)

Because all media services attach to the **`hub_net`** Swarm overlay network, they communicate internally using their **Swarm service names** (`media_<service>`) rather than host ports.

### 6.1 Internal Swarm Service Addresses & Root Folder Mappings

| App | Internal `hub_net` URL | Container Media / Download Paths | Mapped Host / SMB Paths |
| :--- | :--- | :--- | :--- |
| **qBittorrent** | `http://media_qbittorrent:8080` | `/win_torrents`, `/data/torrents` | `/mnt/win_1tb/torrents`, `/mnt/media_drive/data/torrents` |
| **FlareSolverr** | `http://media_flaresolverr:8191` | — | — |
| **Prowlarr** | `http://media_prowlarr:9696` | — | — |
| **Radarr** | `http://media_radarr:7878` | `/win_movies`, `/desktop_movies`, `/data/media/movies`, `/win_torrents` | `/mnt/win_1tb/Movies`, `/mnt/desktop_movies`, `/mnt/media_drive/data/media/movies`, `/mnt/win_1tb/torrents` |
| **Sonarr** | `http://media_sonarr:8989` | `/win_tv_shows`, `/desktop_tv_shows`, `/data/media/tv`, `/win_torrents` | `/mnt/win_1tb/TV_Shows`, `/mnt/desktop_tv_shows`, `/mnt/media_drive/data/media/tv`, `/mnt/win_1tb/torrents` |
| **Jellyseerr** | `http://media_jellyseerr:5055` | — | — |
| **Jellyfin** | `http://media_jellyfin:8096` | `/data/win_movies`, `/data/desktop_movies`, `/data/win_tv_shows`, `/data/desktop_tv_shows`, `/data/media` | `/mnt/win_1tb/Movies`, `/mnt/desktop_movies`, `/mnt/win_1tb/TV_Shows`, `/mnt/desktop_tv_shows`, `/mnt/media_drive/data/media` |
| **Stash** | `http://media_stash:9999` | `/desktop_001`, `/data`, `/generated` | `/mnt/desktop_001`, `/mnt/media_drive/data`, `~/media-stack/config/stash/generated` |

---

### 6.2 Step-by-Step App Wiring (qBittorrent -> Prowlarr -> Radarr/Sonarr -> Seerr -> Jellyfin)

#### Step 1: Configure qBittorrent (`https://torrent.coinflow.lk`)
1. Retrieve the initial temporary password from container logs if logging in for the first time:
   ```bash
   docker service logs media_qbittorrent | grep -i "temporary password"
   ```
2. In **Tools -> Options -> Web UI**, set your username and password, or enable `Bypass authentication for clients in whitelisted IP subnets` (`0.0.0.0/0` is pre-configured in `qBittorrent.conf` behind Cloudflare Access).
3. In **Tools -> Options -> Downloads**:
   - **Default Save Path:** `/win_torrents`
   - **Keep incomplete torrents in:** `/win_torrents/incomplete`
4. In **Tools -> Options -> Speed**, enable bandwidth scheduling so heavy downloads run during off-peak hours.

#### Step 2: Connect Prowlarr to FlareSolverr & Radarr/Sonarr (`https://prowlarr.coinflow.lk`)
1. **Add FlareSolverr Proxy (for Cloudflare-protected indexers):**
   - Go to **Settings -> Indexers -> Add (`+`) -> FlareSolverr**.
   - **Name:** `FlareSolverr`
   - **Tags:** `flaresolverr`
   - **Host:** `http://media_flaresolverr:8191`
   - Add your torrent indexers (e.g., 1337x, YTS, TorrentGalaxy) and apply the `flaresolverr` tag to any indexer protected by Cloudflare.
2. **Sync Indexers with Radarr & Sonarr:**
   - Go to **Settings -> Apps -> Applications -> Add (`+`)**:
   - **For Radarr:**
     - **Prowlarr Server:** `http://media_prowlarr:9696`
     - **Radarr Server:** `http://media_radarr:7878`
     - **API Key:** `<RADARR_API_KEY_PLACEHOLDER>` *(Copy from Radarr -> Settings -> General)*
   - **For Sonarr:**
     - **Prowlarr Server:** `http://media_prowlarr:9696`
     - **Sonarr Server:** `http://media_sonarr:8989`
     - **API Key:** `<SONARR_API_KEY_PLACEHOLDER>` *(Copy from Sonarr -> Settings -> General)*

#### Step 3: Configure Radarr (`https://radarr.coinflow.lk`) & Sonarr (`https://sonarr.coinflow.lk`)
1. **Add Root Folders (`Settings -> Media Management -> Root Folders`):**
   - **In Radarr:** Add `/win_movies` (1 TB BitLocker HDD), `/desktop_movies` (Windows Desktop SMB share), and `/data/media/movies` (SSD staging).
   - **In Sonarr:** Add `/win_tv_shows` (1 TB BitLocker HDD), `/desktop_tv_shows` (Windows Desktop SMB share), and `/data/media/tv` (SSD staging).
2. **Add Download Client (`Settings -> Download Clients -> Add -> qBittorrent`):**
   - **Host:** `media_qbittorrent`
   - **Port:** `8080`
   - **Username / Password:** `<QBIT_CREDENTIALS_PLACEHOLDER>`
   - **Category:** `movies` (in Radarr) / `tv` (in Sonarr)
3. **Remote Path Mapping (`Settings -> Download Clients -> Remote Path Mappings`):**
   - Because both qBittorrent and Radarr/Sonarr mount `/mnt/win_1tb/torrents` at `/win_torrents` and `/mnt/media_drive/data` at `/data`, paths match 1:1 automatically! If qBittorrent saves to `/win_torrents`, ensure `/win_torrents` is mounted in Radarr and Sonarr (already included in `media-stack.yml`).

#### Step 4: Configure Jellyfin (`https://tv.coinflow.lk`)
1. Complete the initial setup wizard and add Media Libraries under **Dashboard -> Libraries -> Add Media Library**:
   - **Movies Library:** Add folders `/data/win_movies`, `/data/desktop_movies`, and `/data/media/movies`.
   - **Shows Library:** Add folders `/data/win_tv_shows`, `/data/desktop_tv_shows`, and `/data/media/tv`.
2. Under **Dashboard -> Networking**, leave Known Proxies / Base URL clean so Nginx Proxy Manager handles reverse proxying transparently.

#### Step 5: Connect Jellyseerr / Seerr (`https://request.coinflow.lk`)
1. Open `https://request.coinflow.lk` and choose **Sign in with Jellyfin**:
   - **Jellyfin URL:** `http://media_jellyfin:8096`
   - Sync your Jellyfin `Movies` and `Shows` libraries.
2. **Connect Radarr:**
   - **Server Name:** `Radarr`
   - **Hostname / IP:** `media_radarr` | **Port:** `7878`
   - **API Key:** `<RADARR_API_KEY_PLACEHOLDER>`
   - **Quality Profile:** `HD-1080p` (or `Any`) | **Root Folder:** `/win_movies`
3. **Connect Sonarr:**
   - **Server Name:** `Sonarr`
   - **Hostname / IP:** `media_sonarr` | **Port:** `8989`
   - **API Key:** `<SONARR_API_KEY_PLACEHOLDER>`
   - **Quality Profile:** `HD-1080p` (or `Any`) | **Root Folder:** `/win_tv_shows`

---

### 6.3 Stash Setup & Security Tripwire Reset (`https://xx.coinflow.lk`)
1. Access `https://xx.coinflow.lk` from your home Wi-Fi/LAN and configure library paths `/desktop_001` and `/data`, with generated metadata pointed to `/generated`.
2. If Stash ever triggers its public internet security lock (`security_tripwire_accessed_from_public_internet`), reset it by editing `~/media-stack/config/stash/config.yml`:
   - Remove the `security_tripwire_accessed_from_public_internet` line and configure authentication in **Settings -> Security**, then restart the service:
     ```bash
     docker service update --force media_stash
     ```

---

### 6.4 Media Library Maintenance & Automated `.zip` Unpack Script

To automatically extract `.zip` movie archives inside `/mnt/win_1tb/Movies`, flatten nested video folders so Jellyfin detects them cleanly, and delete the `.zip` archive once verified:

```bash
cd /mnt/win_1tb/Movies
find . -mindepth 1 -maxdepth 1 -type d | while read -r folder; do
    if [ "$folder" = "./Watched" ]; then continue; fi
    echo "Checking: $folder"

    # 1. Extract zip if no video exists in the main folder
    zipfile=$(find "$folder" -maxdepth 1 -name "*.zip" | head -n 1)
    if [ -n "$zipfile" ] && ! find "$folder" -maxdepth 1 \( -name "*.mkv" -o -name "*.mp4" \) | grep -q .; then
        unzip -qo "$zipfile" -d "$folder"
    fi

    # 2. Move nested videos up to the main movie folder
    find "$folder" -mindepth 2 \( -name "*.mkv" -o -name "*.mp4" \) | while read -r vid; do
        mv "$vid" "$folder/"
    done

    # 3. Delete zip file after confirming video is in the main folder
    if [ -n "$zipfile" ] && find "$folder" -maxdepth 1 \( -name "*.mkv" -o -name "*.mp4" \) | grep -q .; then
        rm "$zipfile"
    fi

    # 4. Clean up empty inner subfolders
    find "$folder" -mindepth 1 -type d -empty -delete
done
```

---

## Part 7: Edge AI Stack, Model Downloads, Hermes Agent & `ask` CLI

### 7.1 Pulling Ollama Models & Off-Peak Nightly Scheduled Downloads

1. **Pull Local LLM Models Immediately:**
   ```bash
   # Pull Gemma 4 (2B) into the running Ollama Swarm container
   docker exec -it $(docker ps -q -f name=ai_stack_ollama) ollama pull gemma4:e2b

   # Verify installed models
   docker exec -it $(docker ps -q -f name=ai_stack_ollama) ollama list
   ```
2. **Schedule Off-Peak Model Downloads (00:05 AM via `systemd-run`):**
   To download large models overnight without slowing down daytime bandwidth:
   ```bash
   systemd-run --on-calendar="*-*-* 00:05:00" --unit=ollama-night-pull \
     /usr/bin/docker exec $(docker ps -q -f name=ai_stack_ollama) sh -c "ollama pull gemma4:e2b && ollama pull hermes3:3b"
   ```

---

### 7.2 NousResearch Hermes Agent Installation & Cloudflare Service Token Auth

1. **Install Hermes Agent Natively on RHEL 10.1:**
   ```bash
   curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
   hermes setup
   hermes --version
   ```
2. **Configure `~/.hermes/config.yaml` with Cloudflare Service Token Headers:**
   ```yaml
   model:
     provider: custom
     default: gemma4:e2b
     base_url: "https://ollama.coinflow.lk/v1"
     extra_headers:
       CF-Access-Client-Id: "<CF_SERVICE_TOKEN_CLIENT_ID_PLACEHOLDER>"
       CF-Access-Client-Secret: "<CF_SERVICE_TOKEN_CLIENT_SECRET_PLACEHOLDER>"
   ```
3. **Start Interactive Agent Session:**
   ```bash
   hermes chat
   ```

---

### 7.3 Terminal CLI Assistant (`ask` + `glow`) Setup on RHEL 10.1

1. **Install `glow` Markdown Renderer via Charm YUM Repository:**
   ```bash
   sudo tee /etc/yum.repos.d/charm.repo > /dev/null <<'EOF'
   [charm]
   name=Charm
   baseurl=https://repo.charm.sh/yum/
   enabled=1
   gpgcheck=1
   gpgkey=https://repo.charm.sh/apt/gpg.key
   EOF

   sudo dnf makecache
   sudo dnf install -y glow
   ```
2. **Configure Provider Variables in `~/.bashrc` (Using Placeholders):**
   ```bash
   export ASK_PROVIDER="gemini"                # Options: "gemini" or "deepseek"
   export GEMINI_API_KEY="<GEMINI_API_KEY_PLACEHOLDER>"
   export GEMINI_MODEL="gemini-3.5-flash-lite"

   export DEEPSEEK_API_KEY="<DEEPSEEK_API_KEY_PLACEHOLDER>"
   export DEEPSEEK_MODEL="deepseek-chat"

   export PATH="$HOME/.local/bin:$PATH"
   ```
   *(Note: Do **not** add `alias ask='nocorrect ask'` in `~/.bashrc` on RHEL 10 — `nocorrect` is a Zsh-only builtin.)*
3. **Run Terminal Queries:**
   ```bash
   ask --status
   ask "check disk space on all mounted filesystems"
   ask -e "default package installer in rhel 10 like apt in ubuntu"
   ```
