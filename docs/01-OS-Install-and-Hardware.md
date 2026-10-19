# 01 — RHEL 10.1 OS Installation, UEFI Bootloader, Wi-Fi Driver Fix & Docker Engine

This guide covers the complete bare-metal installation of **Red Hat Enterprise Linux 10.1** onto an external USB NVMe SSD (`/dev/sda`) on the **`syntax`** laptop (`192.168.1.3`), including **every real-world error encountered during installation, how to prevent them upfront, and how to fix them if they occur**.

---

## 1. Flash Bootable USB & Anaconda Installation

1. **Flash the Installer ISO:**
   - Download the **RHEL 10.1 DVD ISO** (recommended over the small Boot ISO so local packages/RPMs are available offline if needed) and flash it to a USB drive using **Rufus** or **BalenaEtcher**.
2. **Anaconda Partitioning Target (`/dev/sda` External USB NVMe Only):**
   - Select **Installation Destination** $\rightarrow$ choose **ONLY** the external USB NVMe SSD (`/dev/sda`).
   - **Leave internal Windows drives untouched:** Do **not** select `/dev/nvme0n1` (Windows C: OS drive) or `/dev/sdb` (1 TB BitLocker DATA drive).
   - **Preserve `/dev/sda2`:** Keep the existing 50 GB NTFS backup partition (`/dev/sda2`) intact and allocate the remaining ~68 GB to RHEL (`sda3` `/boot/efi`, `sda4` `/boot`, and `sda5` LVM Volume Group `rhel_syntax`).
   - > **💡 Pro Tip to Avoid Later LVM Resizing:** By default, Anaconda splits the LVM Volume Group into a separate `/home` logical volume (~20 GB) and `/` root (~42 GB). Because Docker stores container images in `/var/lib/docker` on `/`, **delete the separate `/home` mount point during Anaconda manual partitioning** and give 100% of the LVM space directly to `/` (62 GB XFS). *(If you used automatic partitioning, see **[02-Storage-BitLocker-and-SMB.md](./02-Storage-BitLocker-and-SMB.md)** for how to reclaim `/home` into `/` online).*
3. **Anaconda Terminal Shortcut:**
   - Press `Ctrl + Alt + F2` at any time during graphical setup to switch to the root TTY shell (`Ctrl + Alt + F6` returns to the graphical installer).

---

## 2. Error #1: Laptop Fails to Boot From External USB NVMe After Install (`BOOTX64.EFI`)

### Why This Error Happened
Standard PC/laptop UEFI firmware looks for removable USB drive bootloaders at the fallback path **`/boot/efi/EFI/BOOT/BOOTX64.EFI`**. However, the RHEL Anaconda installer writes its GRUB/Shim bootloader into **`/boot/efi/EFI/redhat/`** and registers an NVRAM entry that many laptop BIOS implementations drop or ignore for external USB drives.

### How to Fix It (During Install or From Rescue TTY)
Before rebooting out of the Anaconda installer (press `Ctrl + Alt + F2` and run inside `/mnt/sysimage`), or after booting into rescue mode, copy Red Hat's signed shim bootloader to the standard removable EFI path:

```bash
# If inside Anaconda installer TTY (Ctrl+Alt+F2), chroot first:
# chroot /mnt/sysimage

mkdir -p /boot/efi/EFI/BOOT/
cp -rf /boot/efi/EFI/redhat/* /boot/efi/EFI/BOOT/
cp -f /boot/efi/EFI/redhat/shimx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI
```

### Automate Persistence Across Future Kernel Updates
So future `dnf update kernel` upgrades never break USB booting, create a post-install kernel hook:

```bash
sudo tee /etc/kernel/postinst.d/99-sync-usb-efi > /dev/null << 'EOF'
#!/bin/sh
cp -rf /boot/efi/EFI/redhat/* /boot/efi/EFI/BOOT/ 2>/dev/null
cp -f /boot/efi/EFI/redhat/shimx64.efi /boot/efi/EFI/BOOT/BOOTX64.EFI 2>/dev/null
EOF

sudo chmod +x /etc/kernel/postinst.d/99-sync-usb-efi
```

---

## 3. Error #2: Wi-Fi Worked During Install, But No Wi-Fi / No Internet After First Reboot!

### Why This Error Happened
During the graphical Anaconda installation, the live installer environment loads `NetworkManager-wifi` and `wpa_supplicant` in RAM, allowing you to connect to your Wi-Fi network.
However, when you choose **Minimal Install** (or **Server** without wireless groups), Anaconda **does NOT install the `NetworkManager-wifi` and `wpa_supplicant` packages onto the target disk**!
When the server reboots into RHEL 10.1:
- `nmcli device status` shows `wlo1` as `unmanaged` or `unavailable` (missing Wi-Fi plugin).
- You have no internet connection to run `dnf install NetworkManager-wifi`, and you cannot SSH in yet.

### How to Prevent This Next Time (During Install)
- **Prevention Option A:** Plug in a wired **USB-to-Ethernet adapter** (`enp0s20f0u2u1`) or **Android USB Tethering** during first boot so you have immediate wired internet.
- **Prevention Option B:** If using the RHEL Full DVD ISO, select the **Hardware Support** / **NetworkManager Submodules** add-on in Software Selection.

### How to Fix It After Reboot (Installing Wi-Fi Drivers Manually)

#### Method A: Temporarily Plug in USB-Ethernet or Phone USB Tethering (Easiest)
1. Plug an Ethernet cable or connect your phone via USB and turn on **USB Tethering**.
2. Verify wired connectivity and register RHEL subscription repositories:
   ```bash
   ip a
   sudo subscription-manager register --username <RHN_USERNAME_PLACEHOLDER> --password <RHN_PASSWORD_PLACEHOLDER>
   sudo subscription-manager refresh
   ```
3. Install the missing Wi-Fi plugin, `wpa_supplicant`, and Intel wireless firmware:
   ```bash
   sudo dnf install -y NetworkManager-wifi wpa_supplicant iwlwifi-mvm-firmware iwlwifi-dvm-firmware linux-firmware
   sudo systemctl restart NetworkManager
   ```

#### Method B: Completely Offline Fix From the RHEL Installation USB (No Ethernet/Tethering)
If you have no wired internet at all, mount your RHEL 10 DVD ISO USB stick and install the RPMs directly from the USB's `BaseOS`/`AppStream` folders:
```bash
# 1. Check USB partition name (e.g., /dev/sdc1)
lsblk

# 2. Mount the RHEL installer USB
sudo mkdir -p /mnt/usb
sudo mount /dev/sdc1 /mnt/usb

# 3. Install Wi-Fi packages directly from the USB RPM directories
sudo dnf install -y --disablerepo="*" \
  /mnt/usb/BaseOS/Packages/NetworkManager-wifi-*.rpm \
  /mnt/usb/AppStream/Packages/wpa_supplicant-*.rpm

# 4. Restart NetworkManager and unmount USB
sudo systemctl restart NetworkManager
sudo umount /mnt/usb
```

### Connect to Wi-Fi via `nmcli` & Verify Static IP (`192.168.1.3`)
Once `NetworkManager-wifi` is installed and `NetworkManager` is restarted:
```bash
# 1. Scan and connect to your home Wi-Fi
sudo nmcli device wifi rescan
sudo nmcli device wifi list
sudo nmcli --ask device wifi connect "Syntax_SLT_Fibre"

# 2. Verify active connection and IP address (192.168.1.3)
nmcli connection show --active
ip -4 a show wlo1

# 3. Ensure OpenSSH server is running so you can now SSH in from your PC!
sudo systemctl enable --now sshd
```

---

## 4. Register Repositories, EPEL 10 & Base Admin Packages

Once connected via SSH (`ssh sandun@192.168.1.3`), configure your system repositories and install essential administration tools:

```bash
# 1. Enable CodeReady Builder (CRB) and install EPEL 10 for RHEL 10
sudo subscription-manager repos --enable codeready-builder-for-rhel-10-$(arch)-rpms
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm

# 2. Install core CLI utilities, network tools, and system monitors
sudo dnf install -y git curl wget vim nano tmux htop btop jq tar unzip zip \
  net-tools bind-utils lsof tcpdump nmap-ncat bash-completion dnf-plugins-core \
  dnf-automatic libatomic
```

---

## 5. Hardware Hardening: 24/7 Lid-Closed Operation, SSD Trim & KVM

### 5.1 Screen Blanking for 24/7 Lid-Closed Laptop Operation (`~/screenoff.md`)
To run the laptop 24/7 with the lid closed without overheating the LCD backlight or suspending the OS:

```bash
# 1. Blank screen immediately on TTY1
sudo setterm --term linux --blank force </dev/tty1 >/dev/tty1

# 2. Wake screen immediately (if inspecting physically)
sudo setterm --term linux --blank poke </dev/tty1 >/dev/tty1

# 3. Persist 60-second automatic console blanking across all reboots via GRUB kernel cmdline
sudo grubby --update-kernel=ALL --args="consoleblank=60"
```

*(Optional: If your laptop suspends when closing the lid, set `HandleLidSwitch=ignore` and `HandleLidSwitchExternalPower=ignore` in `/etc/systemd/logind.conf` and run `sudo systemctl restart systemd-logind`.)*

### 5.2 Enable Weekly NVMe SSD TRIM
```bash
sudo systemctl enable --now fstrim.timer
systemctl status fstrim.timer --no-pager
```

### 5.3 Install KVM / Libvirt Virtualization (Cockpit Machines)
```bash
sudo dnf install -y qemu-kvm libvirt virt-install cockpit-machines
sudo systemctl enable --now libvirtd cockpit.socket
```

---

## 6. Docker CE Installation, Custom Daemon DNS & Swarm Init

### 6.1 Remove Conflicting Podman / Buildah Packages
RHEL ships with `podman` and `buildah` by default, which conflict with official `docker-ce`:
```bash
sudo dnf remove -y podman buildah runc containers-common
```

### 6.2 Add Official Docker CE Repository & Install Docker 27+
```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add user sandun to the docker group
sudo usermod -aG docker sandun
```

### 6.3 Configure `/etc/docker/daemon.json` (Log Rotation + Custom DNS)
> **⚠️ Important Gotcha:** When `systemd-resolved`'s stub listener (`127.0.0.53`) is disabled on the host for AdGuard Home, Docker containers can fail to resolve external domains if they inherit `127.0.0.1` inside their own network namespace.
> Explicitly configure `dns` in `/etc/docker/daemon.json` with your local AdGuard Home IP (`192.168.1.3`), Router DNS (`192.168.1.1`), and public Cloudflare fallback (`1.1.1.1`):

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json > /dev/null << 'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "dns": ["192.168.1.3", "192.168.1.1", "1.1.1.1"]
}
EOF

sudo systemctl enable --now docker
sudo systemctl restart docker
```

### 6.4 Configure Firewalld & Initialize Docker Swarm
```bash
# Open Docker Swarm cluster ports, DNS (53), and Cockpit (9090), and trust docker_gwbridge
sudo firewall-cmd --permanent --add-port={2377/tcp,7946/tcp,7946/udp,4789/udp}
sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --permanent --add-service=cockpit
sudo firewall-cmd --permanent --zone=trusted --add-interface=docker_gwbridge
sudo firewall-cmd --reload

# Initialize single-node Swarm cluster and create shared attachable overlay network
docker swarm init --advertise-addr 192.168.1.3
docker network create --driver overlay --attachable hub_net
docker network ls | grep hub_net
```

---

➡️ **Next Step:** Proceed to **[02-Storage-BitLocker-and-SMB.md](./02-Storage-BitLocker-and-SMB.md)** to configure UUID lookups, LVM root expansion, automated BitLocker unlocking, and Windows SMB automounts.
