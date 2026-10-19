# 🖥️ RHEL 10.1 Edge AI & Cloud-Native Home Lab (`syntax`)

![OS](https://img.shields.io/badge/OS-RHEL_10.1-EE0000?style=for-the-badge&logo=redhat&logoColor=white)
![Orchestration](https://img.shields.io/badge/Docker_Swarm-27+-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Zero Trust](https://img.shields.io/badge/Cloudflare-Zero_Trust-F38020?style=for-the-badge&logo=cloudflare&logoColor=white)
![Edge AI](https://img.shields.io/badge/AI-Ollama_%26_Hermes-000000?style=for-the-badge&logo=ollama&logoColor=white)
![DNS](https://img.shields.io/badge/DNS-AdGuard_Home-68BC71?style=for-the-badge&logo=adguard&logoColor=white)

> **Production-grade, self-hosted Edge AI, Media Automation, and Zero-Trust Networking cluster running 24/7 on a repurposed laptop booted from an external USB NVMe SSD with Red Hat Enterprise Linux 10.1 and Docker Swarm.**

---

## 🌟 Project Overview

This repository contains the live Docker Swarm stack specifications and service configurations for **`syntax`** (`coinflow.lk`) — a continuous 24/7 home server designed around enterprise Linux administration, zero-trust security, and resource-constrained edge computing.

Instead of leaving internal Windows drives modified or exposing inbound router ports (`80`/`443`/`22`), the entire server boots externally from a dedicated USB NVMe SSD, unlocks encrypted local storage automatically at boot, mounts remote workstation shares on-demand with fault tolerance, and routes traffic globally through **Cloudflare Zero Trust Tunnels** combined with **AdGuard Home Split-DNS** on the local network.

### Key Engineering Highlights
- **Zero-Inbound-Port Security:** Outbound-only `cloudflared.service` tunnel handling wildcard HTTPS (`*.coinflow.lk`) and SSH (`ssh.coinflow.lk` via `ProxyCommand`), backed by Cloudflare Service Tokens for CLI/API authentication.
- **Split-Horizon DNS & IPv6 Leak Prevention:** AdGuard Home rewrites `*.coinflow.lk` to the local LAN IP (`192.168.1.3`) for zero-latency gigabit LAN streaming while blocking `AAAA` queries (`||coinflow.lk^$dnstype=AAAA`) to prevent client IPv6 DNS bypass.
- **Cloudflare ToS-Compliant Media Proxy:** Custom Nginx Proxy Manager location rules detect the `CF-Connecting-IP` header to block video streaming over the public tunnel while allowing full catalog browsing remotely and unrestricted streaming over home Wi-Fi.
- **Edge AI & Agentic CLI:** CPU-throttled (`OLLAMA_NUM_THREADS=3`) local LLM inference (`gemma4:e2b`) via Ollama & Open WebUI, integrated with **NousResearch Hermes Agent** and a custom `/usr/local/bin/ask` + `glow` terminal assistant.
- **Hybrid 3-Tier Storage Architecture:**
  1. **External USB NVMe (`/dev/sda`):** RHEL 10.1 LVM root (`62 GB` XFS after online `/home` reclamation) + `50 GB` NTFS staging partition (`/mnt/media_drive`).
  2. **Internal 1 TB BitLocker HDD (`/dev/sdb1`):** Auto-unlocked at boot via `/etc/crypttab` (`bitlk`) and mounted read-write at `/mnt/win_1tb`.
  3. **Fault-Tolerant Network SMB Shares (`//192.168.1.2/*`):** On-demand `x-systemd.automount` CIFS shares exposed into Swarm containers using `propagation: rshared`.
- **Git Allowlist Backup Pattern:** Strict `.gitignore` (`/*` default ignore with explicit `!/...` allowlisting) tracks live configurations directly from `/home/sandun` without leaking keys, SSH credentials, or container databases.

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

## 📂 Repository Structure

```text
.
├── .gitignore                 # Allowlist-based Git filter for /home/sandun
├── README.md                  # Architecture overview & project showcase
├── guide.md                   # Step-by-step build, deployment & remote access guide
├── ai-iot-stack.yml           # Ollama & Open-WebUI Swarm stack
├── homepage-stack.yml         # Homepage dashboard Swarm stack
├── media-stack.yml            # Jellyfin, Seerr, *arr, qBittorrent, FlareSolverr & Stash stack
├── npm-stack.yml              # Nginx Proxy Manager & AdGuard Home edge stack
├── portainer-stack.yml        # Portainer CE & global Swarm agent stack
├── homepage/
│   └── config/                # Tracked Homepage dashboard YAML & glassmorphism CSS configs
│       ├── bookmarks.yaml
│       ├── custom.css
│       ├── docker.yaml
│       ├── services.yaml
│       ├── settings.yaml
│       └── widgets.yaml
└── media-stack/
    └── config/
        └── qbittorrent/
            └── qBittorrent/
                └── qBittorrent.conf  # Bandwidth scheduler & WebUI subnet whitelist config
```

---

## 🚀 Quick Start (Deploying the Stacks)

For the complete bare-metal installation, storage setup, and remote access walkthrough, see **[`guide.md`](./guide.md)**.

```bash
# 1. Initialize Docker Swarm & create the shared attachable overlay network
docker swarm init
docker network create --driver overlay --attachable hub_net

# 2. Deploy all 5 Swarm stacks
docker stack deploy -c portainer-stack.yml portainer
docker stack deploy -c npm-stack.yml edge
docker stack deploy -c ai-iot-stack.yml ai_stack
docker stack deploy -c homepage-stack.yml homepage
docker stack deploy -c media-stack.yml media

# 3. Verify running Swarm services
docker service ls
```

---

## 🔐 Connecting to the Home Server (`guide.md`)

Detailed instructions for connecting to the server both **at home (LAN/Wi-Fi)** and **remotely from anywhere (Cloudflare Zero Trust SSH & Web)** are documented in **[`guide.md`](./guide.md)**, including:
1. **Workstation SSH Proxy & Local Port Forwarding (`ssh syntax`)** via `cloudflared access ssh`
2. **Split-DNS LAN Resolution** via AdGuard Home (`192.168.1.3`)
3. **CLI AI Assistant & Hermes Agent** usage via Cloudflare Service Tokens
4. **Safe Maintenance Procedures** for BitLocker & SMB automount volumes
