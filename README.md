# Arch Linux Installation Guide
### MSI MAG B650 Tomahawk · Ryzen 7800X3D · RTX 4070 Ti · 32 GB RAM
**Target disk:** `/dev/sda` | **Kernels:** linux, linux-lts, linux-zen | **DE:** KDE Plasma (Wayland) | **Boot:** Limine

---

> **Before you begin:**
> - Boot the Arch ISO in UEFI mode (confirm with `ls /sys/firmware/efi/efivars`)
> - Disable Secure Boot temporarily in BIOS (re-enable optionally after setup)
> - Your Windows disk should be on a *separate* disk — do **not** touch it during partitioning
> - Internet connection assumed (ethernet recommended for install)

---

#
###########################################################################
###########################################################################
# PART 1 — INSTALLATION
###########################################################################
###########################################################################
#

##  Pre-Installation Setup

### Set keyboard layout and font
```bash
loadkeys pl
setfont Lat2-Terminus16
```

### Verify UEFI boot
```bash
cat /sys/firmware/efi/fw_platform_size
```

### Connect to the internet
```bash
# Ethernet: usually auto-configured via DHCP
ping ping.archlinux.org

ip link

# Wi-Fi (if needed):
iwctl
  device list
  device wlan0 set-property Powered on
  station wlan0 scan
  station wlan0 get-networks
  station wlan0 connect "SSID"
  <Enter you wifi password>
  exit
```

### Update system clock
```bash
timedatectl
timedatectl set-ntp true
timedatectl status
```

---

##  Disk Partitioning (Btrfs)

### Partitioning strategy
The layout below separates `/` from `/home` at the **subvolume** level (not partition level),
meaning both live on one Btrfs partition but are independently snapshotted. This lets you
rollback the system without touching your personal files.

```
/dev/sda1  →  4GB        EFI System Partition (FAT32)
/dev/sda2  →  REST       Btrfs main partition (root + home)
```

> No separate swap partition — we use **zram** (configured in Part 2).

### Wipe and partition the disk
```bash
# Verify your target disk
lsblk -f
fdisk -l /dev/sda

# Wipe existing partition table
sgdisk --zap-all /dev/sda

# Create partitions
sgdisk -n 1:0:+4G    -t 1:ef00 -c 1:"EFI"   /dev/sda
sgdisk -n 2:0:0      -t 2:8300 -c 2:"ARCH"  /dev/sda

# Verify
lsblk /dev/sda
```

### Format partitions
```bash
# EFI
mkfs.fat -F32 -n EFI /dev/sda1

# Btrfs
mkfs.btrfs -L ARCH -f /dev/sda2
```

---

##  Btrfs Subvolume Layout

### Mount and create subvolumes
```bash
mount /dev/sda2 /mnt

# Create subvolumes — flat layout (recommended for snapper)
btrfs subvolume create /mnt/@           # root
btrfs subvolume create /mnt/@home       # home (NOT snapshotted with root — safe rollbacks)
btrfs subvolume create /mnt/@log        # /var/log (preserve logs across rollbacks)
btrfs subvolume create /mnt/@cache      # /var/cache (exclude from snapshots)
btrfs subvolume create /mnt/@tmp        # /tmp
btrfs subvolume create /mnt/@snapshots  # /.snapshots (snapper stores here)

umount /mnt
```

### Mount subvolumes with optimal options
```bash
# Common Btrfs mount options (optimized for SSD + NVMe)
BTRFS_OPTS="rw,noatime,compress=zstd:1,space_cache=v2,commit=120,ssd"

mount -o ${BTRFS_OPTS},subvol=@           /dev/sda2 /mnt
mkdir -p /mnt/{boot,home,var/log,var/cache,tmp,.snapshots}

mount -o ${BTRFS_OPTS},subvol=@home       /dev/sda2 /mnt/home
mount -o ${BTRFS_OPTS},subvol=@log        /dev/sda2 /mnt/var/log
mount -o ${BTRFS_OPTS},subvol=@cache      /dev/sda2 /mnt/var/cache
mount -o ${BTRFS_OPTS},subvol=@tmp        /dev/sda2 /mnt/tmp
mount -o ${BTRFS_OPTS},subvol=@snapshots  /dev/sda2 /mnt/.snapshots

# Mount EFI
mount /dev/sda1 /mnt/boot
```

---

##  Base System Installation

### Select mirrors (use reflector)
```bash
sudo reflector --country Poland \
               --protocol https \
               --age 12 \
               --latest 10 \
               --sort rate \
               --save /etc/pacman.d/mirrorlist
```

### Install base system
```bash
pacstrap -K /mnt \
  base base-devel linux linux-headers linux-firmware \
  linux-lts linux-lts-headers \
  linux-zen linux-zen-headers \
  amd-ucode \
  btrfs-progs \
  networkmanager \
  reflector \
  vim nano git \
  sudo \
  zstd
```

> `amd-ucode` — CPU microcode for Ryzen 7800X3D (AMD)

---

##  System Configuration

### Generate fstab
```bash
genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab  # Verify — should show all 7 subvolumes + EFI
```

### Chroot into the new system
```bash
arch-chroot /mnt
```

### Timezone & locale
```bash
ln -sf /usr/share/zoneinfo/Europe/Warsaw /etc/localtime
hwclock --systohc

# Locale
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
echo "pl_PL.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf

vim /etc/vconsole.conf
KEYMAP=pl
```

### Hostname
```bash
echo "archbox" > /etc/hostname

cat > /etc/hosts << EOF
127.0.0.1   localhost
::1         localhost
127.0.1.1   archbox.localdomain archbox
EOF
```

### Root password
```bash
passwd
```

---

##  Bootloader — Limine (for Dual Boot with Windows)

Limine is a modern, fast, and simple BIOS/UEFI bootloader.

### Install Limine
```bash
pacman -S limine efibootmgr

# Deploy Limine EFI
# limine bios-install /dev/sda   # BIOS fallback (optional but good practice)
# cp -v /usr/share/limine/limine-uefi-cd.bin /boot/
mkdir -p /boot/EFI/limine
cp /usr/share/limine/BOOTX64.EFI /boot/EFI/limine/

# Register with UEFI
efibootmgr --create \
  --disk /dev/sda \
  --part 1 \
  --label "Limine" \
  --loader '\EFI\limine\BOOTX64.EFI' \
  --unicode
```

### Create Limine config
# Get UUIDs
```bash
ROOT_UUID=$(blkid -s UUID -o value /dev/sda2)
```

```bash
cat > /boot/limine.conf << 'EOF'
# Limine Configuration
timeout: 5
default_entry: 1

/Arch Linux (linux)
    protocol: linux
    path: boot():/vmlinuz-linux
    cmdline: root=UUID=PLACEHOLDER_UUID rootflags=subvol=@ rw quiet splash nvidia_drm.modeset=1 nvidia_drm.fbdev=1
    module_path: boot():/amd-ucode.img
    module_path: boot():/initramfs-linux.img

/Arch Linux (linux-lts)
    protocol: linux
    path: boot():/vmlinuz-linux-lts
    cmdline: root=UUID=PLACEHOLDER_UUID rootflags=subvol=@ rw quiet splash nvidia_drm.modeset=1 nvidia_drm.fbdev=1
    module_path: boot():/amd-ucode.img
    module_path: boot():/initramfs-linux-lts.img

/Arch Linux (linux-zen)
    protocol: linux
    path: boot():/vmlinuz-linux-zen
    cmdline: root=UUID=PLACEHOLDER_UUID rootflags=subvol=@ rw quiet splash nvidia_drm.modeset=1 nvidia_drm.fbdev=1
    module_path: boot():/amd-ucode.img
    module_path: boot():/initramfs-linux-zen.img

/Snapshots

/Windows
    protocol: efi
    path: boot(windows)/EFI/Microsoft/Boot/bootmgfw.efi
EOF
```

# Replace UUID placeholder with actual UUID
```bash
sed -i "s/PLACEHOLDER_UUID/${ROOT_UUID}/g" /boot/limine.conf
```

# Create Limine Hook:
/etc/pacman.d/hooks/99-limine.hook
```bash
[Trigger]
Operation = Install
Operation = Upgrade
Type = Package
Target = limine              

[Action]
Description = Deploying Limine after upgrade...
When = PostTransaction
Exec = /usr/bin/cp /usr/share/limine/BOOTX64.EFI /boot/EFI/limine/
```

---

##  mkinitcpio — initramfs

Creating initramfs is usually not required because mkinitcpio was run on installation kernel package with the pacstrap

```bash
mkinitcpio -P
```

---

##  User Setup

```bash
useradd -m -G wheel,audio,video,storage,optical,games,network -s /bin/bash yourusername
passwd yourusername

# Grant sudo
EDITOR=vim visudo
# Uncomment: %wheel ALL=(ALL:ALL) ALL
```

---

##  Audio (PipeWire)

```bash
pacman -S \
  pipewire pipewire-alsa pipewire-pulse pipewire-jack \
  wireplumber \
  pavucontrol \
  playerctl
```

> Services are user-level — they start automatically after first login.

---

##  Networking

```bash
pacman -S \
  networkmanager \
  network-manager-applet \
  nm-connection-editor \
  iwd \          # Modern Wi-Fi backend
  avahi \        # mDNS/zeroconf
  nss-mdns

systemctl enable avahi-daemon

# Set iwd as Wi-Fi backend for NetworkManager
cat > /etc/NetworkManager/conf.d/wifi-backend.conf << 'EOF'
[device]
wifi.backend=iwd
EOF
```

---

##  Firewall (UFW)

```bash
pacman -S ufw

systemctl enable ufw

# Default policies
ufw default deny incoming
ufw default allow outgoing

# Allow common services
ufw allow ssh
ufw allow 1900/udp   # UPnP/SSDP (KDE Connect, etc.)
ufw allow 5353/udp   # mDNS

# Enable
ufw enable
ufw status verbose
```

---

##  ZRAM Swap

```bash
pacman -S zram-generator

cat > /etc/systemd/zram-generator.conf << 'EOF'
[zram0]
zram-size = ram / 2         # 16 GB zram from 32 GB RAM
compression-algorithm = zstd
swap-priority = 100
fs-type = swap
EOF

# Tune swappiness (lower = use RAM more aggressively)
cat > /etc/sysctl.d/99-swap.conf << 'EOF'
vm.swappiness = 10
vm.vfs_cache_pressure = 50
EOF
```

---

##  Bluetooth

```bash
pacman -S bluez bluez-utils blueman
systemctl enable bluetooth
```

---

##  Essential Services Verification
```bash
systemctl enable NetworkManager
systemctl enable fstrim.timer          # SSD TRIM (weekly)
systemctl enable bluetooth
systemctl enable bluetooth.service   
systemctl enable ufw
```

---

# Rebuild initramfs one final time
```bash
mkinitcpio -P
```

# Exit chroot and unmount
```bash
exit
umount -R /mnt
reboot
```

---

---

#
###########################################################################
###########################################################################
# PART 2 — POST-INSTALLATION
###########################################################################
###########################################################################
#
---

##  First Boot & AUR Helper (paru)

```bash
# Login as your user, then:
sudo pacman -Syu

# Install paru
sudo pacman -S --needed git base-devel
git clone https://aur.archlinux.org/paru.git /tmp/paru
cd /tmp/paru
makepkg -si
```

---

##  NVIDIA Driver Setup

```bash
# Find GPU Family name
lspci -k -d ::03xx

# Install NVIDIA open kernel modules (RTX 4070 Ti supports open drivers)
pacman -S nvidia-open-dkms nvidia-utils lib32-nvidia-utils \
           nvidia-settings vulkan-icd-loader lib32-vulkan-icd-loader

# Prevent nouveau from loading
cat > /etc/modprobe.d/blacklist-nouveau.conf << 'EOF'
blacklist nouveau
options nouveau modeset=0
EOF

# NVIDIA DRM modesetting (required for Wayland)
cat > /etc/modprobe.d/nvidia.conf << 'EOF'
options nvidia_drm modeset=1 fbdev=1
options nvidia NVreg_UsePageAttributeTable=1
options nvidia NVreg_PreserveVideoMemoryAllocations=1
options nvidia NVreg_EnableGpuFirmware=0
EOF

# Enable NVIDIA services for suspend/resume
systemctl enable nvidia-suspend.service
systemctl enable nvidia-hibernate.service
systemctl enable nvidia-resume.service
```

### Pacman hook — rebuild NVIDIA on kernel update
```bash
mkdir -p /etc/pacman.d/hooks

cat > /etc/pacman.d/hooks/nvidia.hook << 'EOF'
[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=linux
Target=linux-lts
Target=linux-zen
Target=nvidia-open-dkms

[Action]
Description=Update NVIDIA module in initcpio
Depends=mkinitcpio
When=PostTransaction
NeedsTargets
Exec=/usr/bin/limine-mkinitcpio'
EOF
```

---

##  KDE Plasma (Wayland)

```bash
pacman -S \
  plasma-meta \
  kde-applications-meta \
  plasma-login-manager \
#  plasma-wayland-session \
#  xorg-xwayland \
#  xdg-desktop-portal-kde \
#  qt6-wayland \
#  qt5-wayland

paru -S plasma-mobile

#systemctl enable plasma-login-manager
systemctl enable plasmalogin.service
systemctl disable sddm.service

> Check if you have nvidia-drm.modeset=1 in cat /proc/cmdline, if not, then add it

systemctl get-default
> if default.target then change to multi-user.target
systemctl set-default multi-user.target

```

---

##  NVIDIA Fine-Tuning (Wayland)

```bash
# Environment variables for Wayland NVIDIA
mkdir -p ~/.config/plasma-workspace/env

cat > ~/.config/plasma-workspace/env/nvidia-wayland.sh << 'EOF'
export LIBVA_DRIVER_NAME=nvidia
export GBM_BACKEND=nvidia-drm
export __GLX_VENDOR_LIBRARY_NAME=nvidia

# Wayland native apps
export MOZ_ENABLE_WAYLAND=1
export ELECTRON_OZONE_PLATFORM_HINT=auto

# VRR / GSYNC support
export __GL_GSYNC_ALLOWED=1
export __GL_VRR_ALLOWED=1
EOF

chmod +x ~/.config/plasma-workspace/env/nvidia-wayland.sh
```

---

##  Snapper — Btrfs Snapshots with Pre/Post System Updates

### Install snapper and snap-pac
```bash
sudo pacman -S snapper snap-pac
```

`snap-pac` automatically creates **pre** and **post** snapshots around every `pacman` transaction.

### Configure snapper for root
```bash
# Create config (snapper will use /.snapshots subvolume we already created)
sudo snapper -c root create-config /

# Verify
sudo snapper -c root list
cat /etc/snapper/configs/root
```

### Tune snapper cleanup policy
```bash
sudo vim /etc/snapper/configs/root

# Set these values:
# TIMELINE_CREATE="no"          # We don't need timeline — snap-pac handles it
# NUMBER_CLEANUP="yes"
# NUMBER_MIN_AGE="1800"
# NUMBER_LIMIT="10"             # Keep last 10 pre/post pairs
# NUMBER_LIMIT_IMPORTANT="10"
```

### Enable snapper services
```bash
sudo systemctl enable --now snapper-cleanup.timer
sudo systemctl enable --now snapper-timeline.timer  # Optional if timeline disabled
```

### Allow user access to snapshots
```bash
sudo snapper -c root set-config ALLOW_USERS=$(whoami)
sudo chmod a+rx /.snapshots
```

---

# Create Hook limine-mkinitcpio-hook:

Automatically update kernel boot entries in /boot/limine.conf whenever kernels are installed, updated, or removed

```bash
paru -S limine-mkinitcpio-hook
```

> Commands:
> - limine-install installs Limine to your EFI system partition.
> - limine-install --fallback installs Limine as the default fallback boot loader.
> - limine-update updates Limine and generates an initramfs or UKI depending on your initramfs generator:
> - - For mkinitcpio: run limine-mkinitcpio instead of mkinitcpio
> - limine-scan detects active EFI boot entries (dual boot) and allows you to easily add them to Limine.
> - limine-entry-tool --remove-os "Windows Boot Manager" removes an OS entry matching the specified name, leaving its bootable files intact.


> Some non-compliant UEFI motherboards (e.g., certain MSI boards) have non-standard or broken EFI implementations. They may not work with efibootmgr or kernel-based UEFI detection. To skip UEFI detection and registration and set Limine as the fallback boot loader at the standard EFI path /boot/EFI/BOOT/BOOTX64.EFI, run:
```bash
# limine-install --skip-uefi --fallback
```

> **Note:** The Windows entry uses `boot(windows)` — Limine will scan for the Windows EFI on another disk automatically. Adjust if needed.

---

##  Snapshot Boot Selection — Limine Snapper Sync

For Limine, we use **limine-snapper-sync** (AUR) which generates boot entries for each snapshot:

```bash
paru -S limine-snapper-sync

# Configure
sudo vim /etc/limine-snapper-sync.conf
# Set LIMINE_CONFIG=/boot/limine.conf
# Set BTRFS_DEVICE=/dev/sda2

# Enable service — regenerates limine.conf after each snapshot change
#sudo systemctl enable --now limine-snapper-sync.timer
sudo systemctl enable --now limine-snapper-sync.service
```

> After this, every snapper snapshot will get its own entry in the Limine boot menu.
> To **rollback**: boot the snapshot, verify it works, then make it the new default subvolume:
> ```bash
> sudo snapper -c root list
> sudo snapper -c root rollback <snapshot_number>
> reboot
> ```

---

##  TRIM Support (SSD Optimization)

```bash
# Verify TRIM is supported
sudo hdparm -I /dev/sda | grep -i trim

# fstrim timer (already enabled — runs weekly)
sudo systemctl status fstrim.timer

# Verify discard=async is in fstab (it should be from our mount options)
grep discard /etc/fstab
```

---

##  mkinitcpio initramfs Hooks configuration

Configure hooks for Btrfs, NVIDIA, and encryption support.

```bash
cat > /etc/mkinitcpio.conf << 'EOF'
MODULES=(btrfs nvidia nvidia_modeset nvidia_uvm nvidia_drm)
BINARIES=()
FILES=()
HOOKS=(base systemd autodetect microcode kms modconf block keyboard sd-vconsole filesystems fsck)
COMPRESSION="zstd"
COMPRESSION_OPTIONS=(-3)
EOF

mkinitcpio -P
```

> Using `systemd` hook set (modern replacement for udev/etc.)
> `kms` hook ensures DRM is loaded early — needed for NVIDIA DRM modesetting.

---

##  AMD CPU Tuning (Ryzen 7800X3D)

```bash
# Install CPU power management
sudo pacman -S power-profiles-daemon ryzen-smu-dkms

# Limit peak power slightly (optional — 7800X3D runs hot under load)
# Ryzen power plans via KDE Power Management (power-profiles-daemon)
sudo systemctl enable --now power-profiles-daemon

# Install CoreCtrl for fine-grained AMD control (AUR)
paru -S corectrl
```

---

##  Pacman Configuration

```bash
sudo vim /etc/pacman.conf

# Uncomment/add:
Color
VerbosePkgLists
ParallelDownloads = 5
ILoveCandy        # Fun progress bar

# Enable multilib (required for Steam/Wine 32-bit):
# Uncomment [multilib] section:
# [multilib]
# Include = /etc/pacman.d/mirrorlist
```

---

##  Reflector — Auto Mirror Updates

```bash
sudo pacman -S reflector

sudo cat > /etc/xdg/reflector/reflector.conf << 'EOF'
--country "Poland"
--protocol https
--age 12
--latest 10
--sort rate
--save /etc/pacman.d/mirrorlist
EOF

sudo systemctl enable --now reflector.timer
```

---

##  Useful Tools

```bash
sudo pacman -S \
  btop \               # System monitors
  fastfetch \          # System info
  bat eza fd ripgrep \ # Modern CLI tools
  p7zip unrar unzip \  # Archive tools
  cups \               # Printing
  cups-pdf \
  cups-filters \
  vlc \                # Media player
  firefox \            # Browser
  dolphin \
  kate \
  ark \
  konsole \
  plasma-nm \
  spectacle \
  kde-gtk-config \     # GTK theme integration with KDE
  gtk3 gtk4 \
  xdg-utils \
  xdg-user-dirs
```

---

## Printer Driver -Brother

```bash
paru -S brother-hl1210w
```

---

##  Gaming Stack

```bash
pacman -S \
  steam \
  lutris \
  wine \
  wine-mono \
  wine-gecko \
  winetricks \
  gamemode \           # CPU performance governor while gaming
  lib32-gamemode \
  gamescope \          # Wayland gaming compositor (Valve)
  mangohud \           # Performance overlay
  lib32-mangohud \
  vulkan-radeon \      # Keep for compatibility
  lib32-vulkan-radeon \
  mesa \
  lib32-mesa \
  vulkan-tools

# Enable GameMode daemon
systemctl --user enable gamemoded   # Run after first login
```

> In Steam: Right-click game → Properties → Launch Options: `gamemoderun %command%`
> For Proton: Enable in Steam → Settings → Compatibility → Enable Steam Play for all games → Use Proton Experimental

---

##  Flatpak

```bash
sudo pacman -S flatpak
# Flatpak remote
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

```

---

##  KDE Plasma Wayland — Final Tweaks

```bash
# Install KDE Discover backend for firmware updates
sudo pacman -S discover packagekit-qt6 fwupd

# Flatpak integration in Discover
sudo pacman -S flatpak-kcm
```

---

##  Security Hardening

```bash
# Kernel hardening sysctl
sudo cat > /etc/sysctl.d/99-security.conf << 'EOF'
# Disable IP forwarding
net.ipv4.ip_forward = 0
net.ipv6.conf.all.forwarding = 0

# Protect against SYN flood
net.ipv4.tcp_syncookies = 1

# Disable ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0

# Protect against time-wait assassination
net.ipv4.tcp_rfc1337 = 1

# Log martian packets
net.ipv4.conf.all.log_martians = 1

# Harden BPF
kernel.unprivileged_bpf_disabled = 1
net.core.bpf_jit_harden = 2

# Restrict dmesg to root
kernel.dmesg_restrict = 1
EOF

sudo sysctl --system
```

---

##  System Update Workflow (with Snapshots)

With `snap-pac` installed, every `pacman` or `paru` update automatically creates pre/post snapshots.

```bash
# Full system update (snapshots happen automatically)
sudo pacman -Syu

# Or with paru (also covers AUR)
paru -Syu

# View snapshots
sudo snapper -c root list

# Manual snapshot (e.g., before risky change)
sudo snapper -c root create -d "Before my experiment" --type single

# Rollback if something breaks
sudo snapper -c root rollback 5   # Rollback to snapshot #5
reboot
```
