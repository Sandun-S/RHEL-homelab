# 04 — Media Stack Deployment & Complete Servarr / Jellyfin / Seerr Wiring

This guide covers deploying `media-stack.yml` on Docker Swarm and **every step required inside each web UI** to connect **qBittorrent**, **FlareSolverr**, **Prowlarr**, **Radarr**, **Sonarr**, **Jellyfin**, **Jellyseerr (Seerr)**, and **Stash** together over the `hub_net` overlay network.

---

## 1. Host Directory Prerequisites & Permissions

> **⚠️ Gotcha #1:** Docker Swarm **refuses to start a task** if a bind-mounted host directory does not exist (unlike `docker compose`, Swarm does not auto-create missing host folders).
> **⚠️ Gotcha #2:** `media_jellyseerr` (`seerr/seerr:latest`) runs as UID `1000` inside the container and will crash with `EACCES: permission denied` if `/home/sandun/media-stack/config/jellyseerr` is owned by `root:root`.

Run these commands on `syntax` before deploying `media-stack.yml`:

```bash
# 1. Create config folders for all media services (including stash/generated)
mkdir -p /home/sandun/media-stack/config/{qbittorrent,prowlarr,radarr,sonarr,jellyseerr,jellyfin,stash/generated}

# 2. Create staging & torrent directories on the 50 GB NTFS SSD (/mnt/media_drive)
sudo mkdir -p /mnt/media_drive/data/{torrents/{movies,tv,incomplete},media/{movies,tv}}

# 3. Create media & torrent directories on the 1 TB BitLocker HDD (/mnt/win_1tb)
sudo mkdir -p /mnt/win_1tb/{Movies,TV_Shows,torrents/{movies,tv,incomplete}}

# 4. Fix ownership to UID/GID 1000 (sandun) so containers never hit permission errors
sudo chown -R 1000:1000 /home/sandun/media-stack/config /mnt/media_drive/data
```

### Deploy the Media Stack
```bash
docker stack deploy -c /home/sandun/media-stack.yml media
docker service ls --filter "name=media"
```

---

## 2. Internal Swarm (`hub_net`) Service Matrix & Folder Mappings

Because all services in `media-stack.yml` attach to the **`hub_net`** overlay network, **never use `localhost` or `192.168.1.3` when connecting one media container to another**. Always use the **Swarm service name** shown below:

| Application | Public / LAN URL | Internal `hub_net` URL (Use Inside App Settings!) | Container Paths | Mapped Host / SMB Paths |
| :--- | :--- | :--- | :--- | :--- |
| **qBittorrent** | `https://torrent.coinflow.lk` | `http://media_qbittorrent:8080` | `/win_torrents`, `/data` | `/mnt/win_1tb/torrents`, `/mnt/media_drive/data` |
| **FlareSolverr** | *(Internal only)* | `http://media_flaresolverr:8191` | — | — |
| **Prowlarr** | `https://prowlarr.coinflow.lk` | `http://media_prowlarr:9696` | `/config` | `~/media-stack/config/prowlarr` |
| **Radarr** | `https://radarr.coinflow.lk` | `http://media_radarr:7878` | `/win_movies`, `/desktop_movies`, `/data`, `/win_torrents` | `/mnt/win_1tb/Movies`, `/mnt/desktop_movies`, `/mnt/media_drive/data`, `/mnt/win_1tb/torrents` |
| **Sonarr** | `https://sonarr.coinflow.lk` | `http://media_sonarr:8989` | `/win_tv_shows`, `/desktop_tv_shows`, `/data`, `/win_torrents` | `/mnt/win_1tb/TV_Shows`, `/mnt/desktop_tv_shows`, `/mnt/media_drive/data`, `/mnt/win_1tb/torrents` |
| **Jellyfin** | `https://tv.coinflow.lk` | `http://media_jellyfin:8096` | `/data/win_movies`, `/data/desktop_movies`, `/data/win_tv_shows`, `/data/desktop_tv_shows`, `/data/media` | `/mnt/win_1tb/Movies`, `/mnt/desktop_movies`, `/mnt/win_1tb/TV_Shows`, `/mnt/desktop_tv_shows`, `/mnt/media_drive/data/media` |
| **Jellyseerr** | `https://request.coinflow.lk` | `http://media_jellyseerr:5055` | `/app/config` | `~/media-stack/config/jellyseerr` |
| **Stash** | `https://xx.coinflow.lk` | `http://media_stash:9999` | `/desktop_001`, `/data`, `/generated` | `/mnt/desktop_001`, `/mnt/media_drive/data`, `~/media-stack/config/stash/generated` |

---

## 3. Step-by-Step UI Configuration & App Interconnections

### Step 1: Configure qBittorrent (`https://torrent.coinflow.lk`)
1. **First-Time Login:**
   If deploying fresh without the tracked `qBittorrent.conf`, get the temporary password from the Swarm logs:
   ```bash
   docker service logs media_qbittorrent 2>&1 | grep -i "temporary password"
   ```
2. **Download Paths (`Tools -> Options -> Downloads`):**
   - **Default Save Path:** `/win_torrents`
   - **Keep incomplete torrents in:** `/win_torrents/incomplete` (Checked)
3. **Web UI Authentication (`Tools -> Options -> Web UI`):**
   - Set your username and password (`<QBIT_USER_PLACEHOLDER>` / `<QBIT_PASS_PLACEHOLDER>`).
   - `Bypass authentication for clients in whitelisted IP subnets` is enabled (`0.0.0.0/0`) in the tracked `qBittorrent.conf` so internal `hub_net` services (Radarr/Sonarr) and Cloudflare Access sessions connect seamlessly.
4. **Bandwidth Scheduler (`Tools -> Options -> Speed`):**
   - Enable **Schedule the use of alternative rate limits** to throttle daytime bandwidth (`1 KB/s` limit during work hours, unrestricted at night).

---

### Step 2: Configure Radarr (`https://radarr.coinflow.lk`) & Sonarr (`https://sonarr.coinflow.lk`)

#### A. Set Up Root Folders (`Settings -> Media Management -> Root Folders -> Add Root Folder`)
- **In Radarr (`https://radarr.coinflow.lk`):**
  1. `/win_movies` *(1 TB BitLocker HDD — Primary Movies library)*
  2. `/desktop_movies` *(Windows Desktop SMB Share — when Desktop PC is online)*
  3. `/data/media/movies` *(50 GB SSD media folder)*
- **In Sonarr (`https://sonarr.coinflow.lk`):**
  1. `/win_tv_shows` *(1 TB BitLocker HDD — Primary TV Series library)*
  2. `/desktop_tv_shows` *(Windows Desktop SMB Share — when Desktop PC is online)*
  3. `/data/media/tv` *(50 GB SSD media folder)*

#### B. Connect qBittorrent as Download Client (`Settings -> Download Clients -> Add (+) -> qBittorrent`)
Configure this in **both** Radarr and Sonarr:
- **Name:** `qBittorrent`
- **Host:** `media_qbittorrent`
- **Port:** `8080`
- **Use SSL:** `Unchecked`
- **Username / Password:** `<QBIT_USER_PLACEHOLDER>` / `<QBIT_PASS_PLACEHOLDER>` *(or leave blank if subnet whitelist is active)*
- **Category:**
  - In Radarr: `radarr` (or `movies`)
  - In Sonarr: `tv-sonarr` (or `tv`)
- Click **Test** and **Save**.

> **💡 Why No Remote Path Mapping Is Needed:**
> In `media-stack.yml`, qBittorrent, Radarr, and Sonarr all mount `/mnt/win_1tb/torrents` to the exact same container path **`/win_torrents`** (and `/mnt/media_drive/data` to **`/data`**). When qBittorrent tells Radarr/Sonarr *"Download completed in `/win_torrents/..."*, Radarr/Sonarr sees that exact file at `/win_torrents/...` and imports/moves it directly into `/win_movies` or `/win_tv_shows`!

#### C. Copy API Keys for Prowlarr & Jellyseerr
- In both Radarr and Sonarr, go to **Settings -> General -> Security** and copy the **API Key** (`<RADARR_API_KEY_PLACEHOLDER>` and `<SONARR_API_KEY_PLACEHOLDER>`).

---

### Step 3: Configure Prowlarr & FlareSolverr (`https://prowlarr.coinflow.lk`)

#### A. Connect FlareSolverr (Cloudflare Bypass for Indexers)
1. In Prowlarr, go to **Settings -> Indexers -> Add (`+`) -> FlareSolverr**.
2. Fill in:
   - **Name:** `FlareSolverr`
   - **Tags:** `flaresolverr`
   - **Host:** `http://media_flaresolverr:8191`
   - **Request Timeout:** `60`
3. Click **Test** and **Save**.

#### B. Add Public Torrent Indexers
1. Go to **Indexers -> Add Indexer (`+`)**.
2. Add your preferred public trackers (e.g., **YTS**, **1337x**, **TorrentGalaxy**, **EZTV**, **Nyaa**).
3. For any site behind Cloudflare protection (such as `1337x`), add the **`flaresolverr`** tag at the bottom of the indexer window so Prowlarr routes requests through `http://media_flaresolverr:8191`.

#### C. Connect Prowlarr to Radarr & Sonarr (`Settings -> Apps -> Applications -> Add (+)`)
1. **Add Radarr:**
   - **Name:** `Radarr`
   - **Sync Level:** `Full Sync`
   - **Prowlarr Server:** `http://media_prowlarr:9696`
   - **Radarr Server:** `http://media_radarr:7878`
   - **API Key:** `<RADARR_API_KEY_PLACEHOLDER>`
   - Click **Test** and **Save**.
2. **Add Sonarr:**
   - **Name:** `Sonarr`
   - **Sync Level:** `Full Sync`
   - **Prowlarr Server:** `http://media_prowlarr:9696`
   - **Sonarr Server:** `http://media_sonarr:8989`
   - **API Key:** `<SONARR_API_KEY_PLACEHOLDER>`
   - Click **Test** and **Save**.
3. Click **Sync App Indexers** on the top toolbar — all your indexers now appear automatically inside Radarr and Sonarr!

---

### Step 4: Configure Jellyfin Media Server (`https://tv.coinflow.lk`)

1. Open `https://tv.coinflow.lk` (or `http://192.168.1.3:8096`) and finish creating your admin user.
2. Go to **Dashboard -> Libraries -> Add Media Library**:
   - **Library 1 — Content Type: `Movies` | Display Name: `Movies`**
     Add these 3 folders (`+`):
     - `/data/win_movies` *(1 TB BitLocker HDD Movies)*
     - `/data/desktop_movies` *(Windows Desktop SMB Movies)*
     - `/data/media/movies` *(50 GB SSD Movies)*
   - **Library 2 — Content Type: `Shows` | Display Name: `TV Shows`**
     Add these 3 folders (`+`):
     - `/data/win_tv_shows` *(1 TB BitLocker HDD TV Shows)*
     - `/data/desktop_tv_shows` *(Windows Desktop SMB TV Shows)*
     - `/data/media/tv` *(50 GB SSD TV Shows)*

---

### Step 5: Configure Jellyseerr / Seerr Request Portal (`https://request.coinflow.lk`)

1. Open `https://request.coinflow.lk`.
2. **Step 1 — Sign in with Jellyfin:**
   - Choose **Use your Jellyfin account**.
   - **Jellyfin URL:** `http://media_jellyfin:8096`
   - Enter your Jellyfin admin username and password.
3. **Step 2 — Sync Jellyfin Libraries:**
   - Click **Sync Libraries**, enable both **`Movies`** and **`TV Shows`**, and click **Continue**.
4. **Step 3 — Add Radarr Server:**
   - Click **Add Radarr Server**:
     - **Default Server:** `Checked`
     - **Server Name:** `Radarr`
     - **Hostname or IP Address:** `media_radarr`
     - **Port:** `7878` | **Use SSL:** `Unchecked`
     - **API Key:** `<RADARR_API_KEY_PLACEHOLDER>`
   - Click **Test** $\rightarrow$ select **Quality Profile** (`HD-1080p` or `Any`) $\rightarrow$ select **Root Folder** (`/win_movies`) $\rightarrow$ **Add Server**.
5. **Step 4 — Add Sonarr Server:**
   - Click **Add Sonarr Server**:
     - **Default Server:** `Checked`
     - **Server Name:** `Sonarr`
     - **Hostname or IP Address:** `media_sonarr`
     - **Port:** `8989` | **Use SSL:** `Unchecked`
     - **API Key:** `<SONARR_API_KEY_PLACEHOLDER>`
   - Click **Test** $\rightarrow$ select **Quality Profile** (`HD-1080p` or `Any`) $\rightarrow$ select **Root Folder** (`/win_tv_shows`) $\rightarrow$ **Add Server**.

Now, any movie or TV series you request on `https://request.coinflow.lk` from your phone or PC is automatically sent to Radarr/Sonarr $\rightarrow$ searched via Prowlarr $\rightarrow$ downloaded by qBittorrent to `/win_torrents` $\rightarrow$ imported into `/win_movies` or `/win_tv_shows` $\rightarrow$ and ready to watch on Jellyfin (`https://tv.coinflow.lk`)!

---

## 4. Stash Setup & Security Tripwire Reset (`https://xx.coinflow.lk`)

1. Connect to Home Wi-Fi/LAN and open `https://xx.coinflow.lk` (or `http://192.168.1.3:9999`).
2. Add media library directories `/desktop_001` and `/data`, and set generated thumbnails/previews path to `/generated`.
3. **Fixing Stash's Public Internet Security Lockout:**
   If Stash detects an external request before the NPM LAN-only filter is applied, it disables access and writes `security_tripwire_accessed_from_public_internet` to `/home/sandun/media-stack/config/stash/config.yml`. To reset:
   - Edit `/home/sandun/media-stack/config/stash/config.yml` and remove the `security_tripwire_accessed_from_public_internet` line.
   - Restart the service:
     ```bash
     docker service update --force media_stash
     ```

---

## 5. Media Library Cleanup & Automated `.zip` Unpack Script

If movies downloaded manually into `/mnt/win_1tb/Movies` contain `.zip` archives or deeply nested subfolders that confuse Jellyfin, run this automated unpack and flatten script:

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

➡️ **Next Step:** Proceed to **[05-AI-Stack-Hermes-and-Dashboard.md](./05-AI-Stack-Hermes-and-Dashboard.md)** to set up Ollama, Open WebUI, model downloads, Hermes Agent, `ask` CLI, and the Homepage dashboard.
