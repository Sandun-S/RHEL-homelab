# 🖥️ RHEL 10.1 Edge AI & Cloud-Native Home Lab (`syntax`)

![OS](https://img.shields.io/badge/OS-RHEL_10.1-EE0000?style=for-the-badge&logo=redhat&logoColor=white)
![Orchestration](https://img.shields.io/badge/Docker_Swarm-27+-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Zero Trust](https://img.shields.io/badge/Cloudflare-Zero_Trust-F38020?style=for-the-badge&logo=cloudflare&logoColor=white)
![Edge AI](https://img.shields.io/badge/AI-Ollama_%26_Hermes-000000?style=for-the-badge&logo=ollama&logoColor=white)
![DNS](https://img.shields.io/badge/DNS-AdGuard_Home-68BC71?style=for-the-badge&logo=adguard&logoColor=white)

> **Production-grade, self-hosted Edge AI, Media Automation, and Zero-Trust Networking cluster running 24/7 on a repurposed laptop booted from an external USB NVMe SSD with Red Hat Enterprise Linux 10.1 and Docker Swarm.**

---

## 🌟 Project Overview

This repository contains the live Docker Swarm stack specifications, service configurations, and complete bare-metal engineering runbooks for **`syntax`** (`coinflow.lk` / `192.168.1.3`) — a continuous 24/7 home server built around enterprise Linux administration, zero-trust security, and resource-constrained edge computing.

Instead of modifying internal Windows drives or exposing inbound router ports (`80`/`443`/`22`), the entire server boots externally from a dedicated USB NVMe SSD, unlocks encrypted local BitLocker storage automatically at boot, mounts remote workstation shares on-demand with fault tolerance, and routes traffic globally through **Cloudflare Zero Trust Tunnels** paired with **AdGuard Home Split-DNS** on the local network.

---

## 📚 Complete Rebuild & Operations Guides (`docs/`)

If you are rebuilding this server from scratch, connecting to it from a new workstation, or wiring up the services, follow the 5 modular guides in **[`docs/`](./docs/)**:

| Guide | What It Covers |
| :--- | :--- |
| **[`01-OS-Install-and-Hardware.md`](./docs/01-OS-Install-and-Hardware.md)** | **RHEL 10.1 Bare-Metal Install & Real-World Error Fixes:** External USB NVMe partitioning, fixing the UEFI `BOOTX64.EFI` removable media boot failure, fixing the missing Wi-Fi driver (`NetworkManager-wifi` + `wpa_supplicant`) after minimal install, 24/7 lid-closed screen blanking, KVM/Cockpit setup, Docker CE installation, and `/etc/docker/daemon.json` DNS (`192.168.1.3`, `192.168.1.1`, `1.1.1.1`). |
| **[`02-Storage-BitLocker-and-SMB.md`](./docs/02-Storage-BitLocker-and-SMB.md)** | **Hybrid 3-Tier Storage Engineering:** How to inspect disks and extract `UUID` / `PARTUUID` via `lsblk -f` and `blkid`, mounting `/dev/sda2` NTFS (`ntfs-3g`), online LVM `/home` removal & XFS root expansion to 62 GB (`lvextend -r`), automated boot-time BitLocker decryption via `/etc/crypttab`, fault-tolerant Windows Desktop SMB `x-systemd.automount` shares, and safe Swarm scale-down before unmounting. |
| **[`03-Cloudflare-NPM-and-AdGuard-DNS.md`](./docs/03-Cloudflare-NPM-and-AdGuard-DNS.md)** | **Zero-Trust Ingress, Client SSH & Split-DNS:** Installing `cloudflared` on both the RHEL server and client workstation, `~/.ssh/config` (`ssh syntax`) with `LocalForward` admin tunnels (`3005`, `8081`, `8082`), Cloudflare Tunnel routes & mobile bypass policies, freeing port 53 (`systemd-resolved`), all 14 Nginx Proxy Manager host rules + custom location blocks, and AdGuard Home Split-DNS with IPv6 `AAAA` leak prevention. |
| **[`04-Media-and-Servarr-Setup.md`](./docs/04-Media-and-Servarr-Setup.md)** | **Media Stack & Complete App Interconnections:** Creating required host bind-mount folders (`UID 1000`), deploying `media-stack.yml`, internal `hub_net` service URLs, and step-by-step UI wiring across **qBittorrent** $\rightarrow$ **FlareSolverr** $\rightarrow$ **Prowlarr** $\rightarrow$ **Radarr & Sonarr** $\rightarrow$ **Jellyfin** $\rightarrow$ **Jellyseerr (Seerr)** $\rightarrow$ **Stash**, plus the automated `.zip` unpack script. |
| **[`05-AI-Stack-Hermes-and-Dashboard.md`](./docs/05-AI-Stack-Hermes-and-Dashboard.md)** | **Edge AI, Model Downloads, Hermes Agent & Homepage:** Deploying `ai-iot-stack.yml` (CPU-throttled Ollama + Open WebUI), pulling `gemma4:e2b` and scheduling off-peak `00:05` nightly downloads via `systemd-run`, installing NousResearch Hermes Agent (`libatomic` + Cloudflare Service Token headers), setting up the `ask` + `glow` CLI assistant, and configuring the Homepage dashboard (`Syntax Edge Hub`). |

---

## 🏗️ System Architecture

```text
                                  [ INTERNET ]
                                        │
                         Cloudflare Tunnel (cloudflared.service)
                         *.coinflow.lk   -> localhost:80
                         ssh.coinflow.lk -> localhost:22
                                        │
┌─────────────────────────────── syntax Host (192.168.1.3) ───────────────────────────────┐
│                                       │                                                 │
│                                       ▼                                                 │
│                        [ Nginx Proxy Manager (edge_npm) ]                               │
│                         Port 80 (HTTP) / Port 81 (Admin) / Port 443 (SSL)               │
│                                       │                                                 │
│       ┌───────────────────────────────┼───────────────────────────────┐                 │
│       │ Overlay: hub_net              │ Overlay: hub_net              │ Host Network    │
│       ▼                               ▼                               ▼                 │
│  [ Open WebUI ]                [ Homepage ]                    [ RHEL Cockpit + KVM ]   │
│  ai_stack_open-webui:8080      homepage_homepage:3000          192.168.1.3:9090         │
│                                                                                         │
│       │ Overlay: hub_net              │ Overlay: portainer_net        │ Overlay: hub_net│
│       ▼                               ▼                               ▼                 │
│  [ Ollama Engine ]             [ Portainer CE + Agent ]        [ Servarr + Stash ]      │
│  ai_stack_ollama:11434         portainer_portainer:9443        media_* (Jellyfin :8096, │
│                                                                Stash :9999, Seerr, *arr,│
│                                                                qBit :6881, FlareSolverr)│
│  [ AdGuard Home (edge_adguard) ]                                                        │
│  - Host Port 53 (UDP/TCP) <- LAN DNS queries (Phones, Workstation, Router)              │
│  - Host Port 3005 -> Container Port 3000 (Web UI)                                       │
│  - DNS Rewrite: *.coinflow.lk -> 192.168.1.3 (AAAA blocked)                             │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 📦 Complete Services & Stack Inventory

All application stacks communicate over the external attachable Swarm overlay network **`hub_net`** (and **`portainer_portainer_net`** for cluster administration):

| Stack | Service / Container | Host Ports | Internal (`hub_net`) | Domain Route | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **System** | Cockpit + KVM | `9090/tcp` | — | `cockpit.coinflow.lk` | Web-based RHEL 10 & KVM VM administration |
| **System** | OpenSSH (`sshd`) | `22/tcp` | — | `ssh.coinflow.lk` | Remote terminal via `cloudflared` SSH proxy |
| **System** | Cloudflared Daemon | Outbound `443` | — | — | Host systemd Zero Trust connector (`cloudflared.service`) |
| **Edge (`npm-stack.yml`)** | `edge_npm` | `80`, `81`, `443` | `80`, `81`, `443` | `npm.coinflow.lk` | Reverse proxy, custom security rules & Let's Encrypt SSL |
| **Edge (`npm-stack.yml`)** | `edge_adguard` | `53/tcp`, `53/udp`, `3005/tcp` | `3000/tcp` | `adguard.coinflow.lk` | Network-wide ad blocker, DoH/DoT & split-DNS |
| **Mgmt (`portainer-stack.yml`)** | `portainer_portainer` | `8000/tcp`, `9443/tcp` | `9443/tcp` | `portainer.coinflow.lk` | Docker Swarm & stack management UI |
| **Mgmt (`portainer-stack.yml`)** | `portainer_agent` | — | `9001/tcp` | — | Global Swarm node inspection agent |
| **Dash (`homepage-stack.yml`)** | `homepage_homepage` | `3001/tcp` | `3000/tcp` | `home.coinflow.lk` | *Syntax Edge Hub* dashboard, disk & Swarm metrics |
| **AI (`ai-iot-stack.yml`)** | `ai_stack_open-webui` | `3000/tcp` | `8080/tcp` | `ai.coinflow.lk` | Local LLM chat interface |
| **AI (`ai-iot-stack.yml`)** | `ai_stack_ollama` | `11434/tcp` | `11434/tcp` | `ollama.coinflow.lk` | Local LLM inference API (`gemma4:e2b`) |
| **Media (`media-stack.yml`)** | `media_jellyfin` | `8096/tcp` (`host`) | `8096/tcp` | `tv.coinflow.lk` | Media streaming server (LAN playback enforced) |
| **Media (`media-stack.yml`)** | `media_jellyseerr` | — | `5055/tcp` | `request.coinflow.lk` | Media discovery & request portal (`seerr/seerr`) |
| **Media (`media-stack.yml`)** | `media_radarr` | — | `7878/tcp` | `radarr.coinflow.lk` | Automated movie library manager |
| **Media (`media-stack.yml`)** | `media_sonarr` | — | `8989/tcp` | `sonarr.coinflow.lk` | Automated TV series library manager |
| **Media (`media-stack.yml`)** | `media_prowlarr` | — | `9696/tcp` | `prowlarr.coinflow.lk` | Centralized torrent indexer proxy |
| **Media (`media-stack.yml`)** | `media_qbittorrent` | `6881/tcp+udp` (`host`) | `8080/tcp` | `torrent.coinflow.lk` | Bandwidth-scheduled torrent client |
| **Media (`media-stack.yml`)** | `media_flaresolverr` | — | `8191/tcp` | *(Internal)* | Cloudflare challenge solver for Prowlarr |
| **Media (`media-stack.yml`)** | `media_stash` | `9999/tcp` (`host`) | `9999/tcp` | `xx.coinflow.lk` | Media organizer & streamer (strictly LAN-locked) |

---

## 🔐 Quick Access Reference (How to Connect at Home or Remotely)

1. **Web Access (Same URLs Everywhere):**
   - **At Home (Wi-Fi/LAN):** AdGuard Home (`192.168.1.3`) rewrites `*.coinflow.lk` directly to `192.168.1.3` over Gigabit LAN (unlocking full video playback on `https://tv.coinflow.lk` and `https://xx.coinflow.lk`).
   - **Away from Home:** Cloudflare Zero Trust routes `*.coinflow.lk` through `cloudflared.service`.
2. **Terminal SSH & Admin Port Forwarding (`~/.ssh/config`):**
   Install `cloudflared` on your client PC ([see Guide 03](./docs/03-Cloudflare-NPM-and-AdGuard-DNS.md#3-client-workstation-setup-how-to-ssh--tunnel-into-the-home-server)) and add:
   ```sshconfig
   Host syntax
       HostName ssh.coinflow.lk
       User sandun
       ProxyCommand ~/.local/bin/cloudflared access ssh --hostname %h
       LocalForward 3005 127.0.0.1:3005
       LocalForward 8081 127.0.0.1:81
       LocalForward 8082 192.168.1.1:80
   ```
   Then run `ssh syntax` (or `ssh sandun@192.168.1.3` directly on LAN).

---

## 🚀 Quick Stack Deployment Commands

```bash
# 1. Create the shared attachable Swarm overlay network
docker network create --driver overlay --attachable hub_net

# 2. Deploy all 5 Swarm stacks
docker stack deploy -c portainer-stack.yml portainer
docker stack deploy -c npm-stack.yml edge
docker stack deploy -c ai-iot-stack.yml ai_stack
docker stack deploy -c homepage-stack.yml homepage
docker stack deploy -c media-stack.yml media
```
