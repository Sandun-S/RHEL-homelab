# 02 — Hybrid 3-Tier Storage, UUID Lookup, LVM Expansion, BitLocker & SMB Automounts

This guide covers the complete storage setup on **`syntax`** (`192.168.1.3`), including **how to inspect disks and find `UUID` / `PARTUUID` values via `lsblk` and `blkid`**, how to reclaim `/home` into `/` online, how to auto-unlock the 1 TB Windows BitLocker drive at boot, and how to mount Windows Desktop SMB shares with fault-tolerant `x-systemd.automount`.

---

## 1. Inspecting Disks & Finding `UUID` / `PARTUUID` Values

> **⚠️ Critical Rule for `/etc/fstab` & `/etc/crypttab`:** Never use device names like `/dev/sda2` or `/dev/sdb1` in `/etc/fstab` or `/etc/crypttab`! USB drives and SATA/NVMe controllers can change letter order (`/dev/sda` $\leftrightarrow$ `/dev/sdb`) across reboots, which will drop RHEL into `emergency.target` boot failure. **Always look up and use `UUID` or `PARTUUID`.**

### 1.1 View Complete Disk & Partition Tree (`lsblk -f`)
Run `lsblk -f` to see every connected disk, its filesystem type (`FSTYPE`), label, `UUID`, and current mountpoint:

```bash
lsblk -f
```

**Expected Output on `syntax`:**
```text
NAME                 FSTYPE      FSVER    LABEL      UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
sda                                                                                                        
├─sda1                                                                                                     
├─sda2               ntfs                 Local Disk 2C66F38766F34FD6                         18.4G    63% /mnt/media_drive
├─sda3               vfat        FAT32               C96A-1967                               582.3M     3% /boot/efi
├─sda4               xfs                             690617a0-9be2-42f4-890e-3f45bc01e3b1    474.1M    51% /boot
└─sda5               LVM2_member LVM2 001            DaGE62-cOiC-xhdP-Em3F-moKf-cWS5-UlIzg9                
  ├─rhel_syntax-root xfs                             561a3293-4169-4272-b707-ba9cdd795979     14.6G    76% /
  └─rhel_syntax-swap swap        1                   1c1a9fe9-11f9-4179-a6e6-bd760551e779                  [SWAP]
sdb                                                                                                        
└─sdb1               BitLocker   2                                                                         
  └─win_1tb          ntfs                 DATA       486814A568149432                        378.1G    59% /mnt/win_1tb
nvme0n1                                                                                                    
├─nvme0n1p1          vfat        FAT32    SYSTEM     522C-E2E7                                             
├─nvme0n1p2                                                                                                
├─nvme0n1p3          BitLocker   2                                                                         
├─nvme0n1p4          ntfs                 RECOVERY   8E421A034219F127                                      
└─nvme0n1p5          vfat        FAT32    MYASUS     5624-7DEF                                             
```

### 1.2 How to Extract Exact `UUID` and `PARTUUID` (`blkid`)
Use `sudo blkid` to inspect partition identifiers before editing `/etc/fstab` or `/etc/crypttab`:

```bash
# List all block devices with UUIDs and PARTUUIDs
sudo blkid

# Get ONLY the filesystem UUID of the 50 GB NTFS partition (/dev/sda2)
sudo blkid -s UUID -o value /dev/sda2
# Output: 2C66F38766F34FD6

# Get ONLY the partition PARTUUID of the 1 TB BitLocker drive (/dev/sdb1)
# (Note: Encrypted BitLocker partitions don't expose an NTFS UUID until unlocked, so crypttab MUST use PARTUUID!)
sudo blkid -s PARTUUID -o value /dev/sdb1
# Output: 7937ddea-af22-4c7e-8660-2510540ef0a7
```

---

## 2. Mount Secondary 50 GB NTFS Partition (`/dev/sda2` $\rightarrow$ `/mnt/media_drive`)

The external USB NVMe SSD has a 50 GB NTFS partition (`/dev/sda2`) used for Windows backups and download staging (`/mnt/media_drive/data`).

```bash
# 1. Install NTFS-3G driver from EPEL
sudo dnf install -y epel-release
sudo dnf install -y ntfs-3g

# 2. Create mount directory
sudo mkdir -p /mnt/media_drive

# 3. Look up the UUID of /dev/sda2 dynamically
SDA2_UUID=$(sudo blkid -s UUID -o value /dev/sda2)
echo "Found UUID: $SDA2_UUID"

# 4. Test manual mount first (uid=1000,gid=1000 grants user 'sandun' ownership)
sudo mount -t ntfs-3g -o uid=1000,gid=1000,umask=022 UUID=$SDA2_UUID /mnt/media_drive
df -h /mnt/media_drive

# 5. Persist in /etc/fstab and verify fstab syntax
echo "UUID=$SDA2_UUID /mnt/media_drive ntfs-3g uid=1000,gid=1000,umask=022 0 0" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
```

---

## 3. Online LVM `/home` Reclamation into `/` (Expanding Root to 62 GB)

### Why This Was Needed
Anaconda's default LVM layout split `/dev/sda5` into `rhel_syntax-root` (~41.6 GB) and `rhel_syntax-home` (~20.3 GB). Pulling Docker images into `/var/lib/docker` filled `/` to **97% (only 1.6 GB free)**. Because XFS filesystems cannot be shrunk, `/home` had to be backed up, unmounted, deleted, and its 20.3 GB extents merged into `/dev/rhel_syntax/root`.

### Step-by-Step Execution (From Root Shell)
> **⚠️ Gotcha:** Do not run this while logged in as `sandun` or while Docker containers are running, or `umount /home` will fail with `target is busy`.

```bash
# 1. Back up /home/sandun to the secondary NTFS partition
sudo cp -a /home/sandun /mnt/media_drive/sandun_backup

# 2. Switch to an interactive root shell
sudo -i

# 3. Stop Docker service AND docker.socket so containers release bind mounts in /home/sandun
systemctl stop docker docker.socket

# 4. Kill any remaining processes owned by 'sandun' holding /home open
pkill -u sandun
fuser -vkm -9 /home

# 5. Unmount /home and remove the logical volume
umount /home
lvremove -y /dev/rhel_syntax/home

# 6. Extend root logical volume by 100% of free space and resize XFS online (-r)
lvextend -r -l +100%FREE /dev/rhel_syntax/root

# 7. Delete the old /home line from /etc/fstab so RHEL doesn't halt on boot!
sed -i '\|/home|d' /etc/fstab
systemctl daemon-reload

# 8. Recreate /home directory on the root partition, restore files, and fix SELinux contexts
mkdir -p /home
cp -a /mnt/media_drive/sandun_backup /home/sandun
chown -R sandun:sandun /home/sandun
restorecon -R /home/sandun

# 9. Restart Docker and verify 62 GB root space
systemctl start docker.socket docker
df -h /
```

---

## 4. Automated Boot-Time BitLocker Unlock (`/dev/sdb1` $\rightarrow$ `/mnt/win_1tb`)

The laptop's internal 1 TB HDD (`/dev/sdb1`) is encrypted with **Windows BitLocker 2**. RHEL 10's `cryptsetup` (`systemd-cryptsetup`) natively supports unlocking BitLocker (`bitlk`) volumes using the 48-digit recovery password (`XXXXXX-XXXXXX-...`).

### 4.1 Test Unlocking Manually First (To Verify Recovery Key)
Before automating in `/etc/crypttab`, verify your BitLocker metadata and recovery password:
```bash
# Inspect BitLocker header on /dev/sdb1
sudo cryptsetup bitlkDump /dev/sdb1

# Test unlocking manually to /dev/mapper/win_1tb
sudo cryptsetup open --type bitlk /dev/sdb1 win_1tb
ls -l /dev/mapper/win_1tb

# Close test mapping before setting up systemd automation
sudo cryptsetup close win_1tb
```

### 4.2 Create Protected Keyfile (`/etc/cryptsetup-keys.d/win_1tb.key`)
```bash
# Create directory (700) and empty key file (600) owned by root:root
sudo install -d -m 700 /etc/cryptsetup-keys.d
sudo install -m 600 /dev/null /etc/cryptsetup-keys.d/win_1tb.key

# Paste ONLY the single-line 48-digit BitLocker recovery password (no trailing spaces)
sudoedit /etc/cryptsetup-keys.d/win_1tb.key

# Confirm strict root:root 600 permissions
sudo stat -c '%U:%G %a %n' /etc/cryptsetup-keys.d/win_1tb.key
```

### 4.3 Configure `/etc/crypttab` and `/etc/fstab`
```bash
# 1. Get the stable PARTUUID of /dev/sdb1
SDB1_PARTUUID=$(sudo blkid -s PARTUUID -o value /dev/sdb1)
echo "Found BitLocker PARTUUID: $SDB1_PARTUUID"

# 2. Add entry to /etc/crypttab
echo "win_1tb PARTUUID=$SDB1_PARTUUID /etc/cryptsetup-keys.d/win_1tb.key bitlk" | sudo tee -a /etc/crypttab

# 3. Create mount point and add unlocked mapper device to /etc/fstab
sudo mkdir -p /mnt/win_1tb
echo "/dev/mapper/win_1tb /mnt/win_1tb ntfs-3g rw,uid=1000,gid=1000,umask=022 0 0" | sudo tee -a /etc/fstab

# 4. Reload systemd generators, trigger unlock service, and mount
sudo systemctl daemon-reload
sudo systemctl start systemd-cryptsetup@win_1tb.service
sudo udevadm settle
sudo mount /mnt/win_1tb

# 5. Verify mount and read-write access
findmnt /mnt/win_1tb
df -h /mnt/win_1tb
```

---

## 5. Windows Desktop SMB Network Shares (`192.168.1.2` $\rightarrow$ `syntax`)

To allow Jellyfin, Radarr, Sonarr, and Stash on `syntax` (`192.168.1.3`) to stream and manage media stored on the Windows Desktop PC (`192.168.1.2`) whenever the desktop is turned on — **without hanging the server if the desktop is turned off**:

### 5.1 Windows Desktop Sharing Configuration (`192.168.1.2`)
1. Right-click each folder (`Movies`, `TV_Shows`, `001`) $\rightarrow$ **Properties** $\rightarrow$ **Sharing** $\rightarrow$ **Advanced Sharing**:
   - Check **Share this folder**.
   - Click **Permissions** $\rightarrow$ Add **`Everyone`** with **Read** and **Change** permissions.
2. Open **Control Panel** $\rightarrow$ **Network and Sharing Center** $\rightarrow$ **Advanced sharing settings**:
   - **Private & Public networks:** Turn **ON** Network Discovery and File & Printer Sharing.
   - **All Networks:** Turn **OFF** **Password protected sharing** (enables guest CIFS access over the home LAN).

### 5.2 RHEL CIFS Automount Configuration (`/etc/fstab`)
1. Install CIFS/Samba client packages and create mount directories:
   ```bash
   sudo dnf install -y cifs-utils samba-client
   sudo mkdir -p /mnt/desktop_movies /mnt/desktop_tv_shows /mnt/desktop_001
   ```
2. Add the fault-tolerant `x-systemd.automount` entries to `/etc/fstab`:
   ```fstab
   # Windows Desktop SMB Shares (Auto-mount on demand, auto-skip in 5s if Desktop PC is OFF)
   //192.168.1.2/Movies    /mnt/desktop_movies    cifs  username=guest,password=,uid=1000,gid=1000,vers=3.0,soft,_netdev,nofail,x-systemd.automount,x-systemd.idle-timeout=60,x-systemd.mount-timeout=5s  0  0
   //192.168.1.2/TV_Shows  /mnt/desktop_tv_shows  cifs  username=guest,password=,uid=1000,gid=1000,vers=3.0,soft,_netdev,nofail,x-systemd.automount,x-systemd.idle-timeout=60,x-systemd.mount-timeout=5s  0  0
   //192.168.1.2/001       /mnt/desktop_001       cifs  username=guest,password=,uid=1000,gid=1000,vers=3.0,soft,_netdev,nofail,x-systemd.automount,x-systemd.idle-timeout=60,x-systemd.mount-timeout=5s  0  0
   ```
3. Reload systemd and trigger automounts:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart remote-fs.target
   ls -la /mnt/desktop_movies /mnt/desktop_tv_shows /mnt/desktop_001
   ```

#### Why `propagation: rshared` is Used in `media-stack.yml`
Because `x-systemd.automount` unmounts the SMB share after 60 seconds of inactivity (`idle-timeout=60`) and re-mounts it on access, standard Docker bind mounts lose visibility of the mounted filesystem unless **`bind.propagation: rshared`** is set on the container volume.

---

## 6. Reference: Complete Live `/etc/fstab` & Safe Unmount Procedure

### Complete `/etc/fstab` on `syntax`
```fstab
UUID=561a3293-4169-4272-b707-ba9cdd795979 /                       xfs     defaults        0 0
UUID=690617a0-9be2-42f4-890e-3f45bc01e2b1 /boot                   xfs     defaults        0 0
UUID=C96A-1967          /boot/efi               vfat    umask=0077,shortname=winnt 0 2
UUID=1c1a9fe9-11f9-4179-a6e6-bd760551e779 none                    swap    defaults        0 0

# Local NTFS & Unlocked BitLocker Drives
UUID=2C66F38766F34FD6 /mnt/media_drive ntfs-3g uid=1000,gid=1000,umask=022 0 0
/dev/mapper/win_1tb   /mnt/win_1tb     ntfs-3g rw,uid=1000,gid=1000,umask=022 0 0

# Windows Desktop SMB Shares (Auto-skip if PC is off)
//192.168.1.2/Movies    /mnt/desktop_movies    cifs  username=guest,password=,uid=1000,gid=1000,vers=3.0,soft,_netdev,nofail,x-systemd.automount,x-systemd.idle-timeout=60,x-systemd.mount-timeout=5s  0  0
//192.168.1.2/001       /mnt/desktop_001       cifs  username=guest,password=,uid=1000,gid=1000,vers=3.0,soft,_netdev,nofail,x-systemd.automount,x-systemd.idle-timeout=60,x-systemd.mount-timeout=5s  0  0
//192.168.1.2/TV_Shows  /mnt/desktop_tv_shows  cifs  username=guest,password=,uid=1000,gid=1000,vers=3.0,soft,_netdev,nofail,x-systemd.automount,x-systemd.idle-timeout=60,x-systemd.mount-timeout=5s  0  0
```

### Safe Unmount / Remount Procedure for `/mnt/win_1tb`
If you ever need to unmount `/mnt/win_1tb`, scale down the Swarm services holding bind mounts first:
```bash
# 1. Scale down services holding /mnt/win_1tb
docker service scale media_jellyfin=0 media_qbittorrent=0 media_radarr=0 media_sonarr=0 media_stash=0

# 2. Unmount and close BitLocker volume
sudo umount /mnt/win_1tb
sudo systemctl stop systemd-cryptsetup@win_1tb.service
sudo cryptsetup close win_1tb
sudo udevadm settle

# 3. Re-unlock, mount, and restore replicas
sudo systemctl start systemd-cryptsetup@win_1tb.service
sudo udevadm settle
sudo mount /mnt/win_1tb
docker service scale media_jellyfin=1 media_qbittorrent=1 media_radarr=1 media_sonarr=1 media_stash=1
```

---

➡️ **Next Step:** Proceed to **[03-Cloudflare-NPM-and-AdGuard-DNS.md](./03-Cloudflare-NPM-and-AdGuard-DNS.md)** to set up Cloudflare Zero Trust tunnels, client SSH access, Nginx Proxy Manager, and AdGuard Split-DNS.
