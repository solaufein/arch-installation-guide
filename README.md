# Arch Linux Installation Guide
### MSI MAG B650 Tomahawk · Ryzen 7800X3D · RTX 4070 Ti · 32 GB RAM
**Target disk:** `/dev/sda` | **Kernels:** linux, linux-lts, linux-zen | **DE:** KDE Plasma (Wayland) | **Boot:** Limine
**Features:** btrfs, zram, snapper, snap-pac, ufw

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

# Wi-Fi (if you dont have LAN internet connection):
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
sudo reflector --protocol https \
               --age 12 \
               --latest 10 \
               --sort age \
               --save /etc/pacman.d/mirrorlist
```

### Install base system
```bash
vim /mnt/etc/vconsole.conf      # We need it before because pacstrap can raise some errors about missing vconsole
KEYMAP=pl

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

# `amd-ucode` — CPU microcode for Ryzen 7800X3D (AMD)
```

---

##  System Configuration

### Generate fstab
```bash
genfstab -U /mnt >> /mnt/etc/fstab

# Verify — should show all 7 subvolumes + EFI
cat /mnt/etc/fstab
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

vim /etc/hosts
127.0.0.1   localhost
::1         localhost
127.0.1.1   archbox.localdomain archbox
```

### Root password
```bash
passwd
```

---

##  Bootloader — Limine (Dual Boot with Windows)
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
### Get UUIDs
```bash
ROOT_UUID=$(blkid -s UUID -o value /dev/sda2)

# Get machine-id for custom comment (optional, useful for identifying entries in multi-boot)
MACHINE_UUID=$(cat /etc/machine-id)

# Get Windows EFI FAT32 parition ID
lsblk -f      # find Windows disk/partition FAT32 label, eg: /dev/nvme0n1p1
WINDOWS_UUID=$(sudo blkid -s PARTUUID -o value /dev/nvme0n1p1)
```

```bash
vim /boot/limine.conf

timeout: 5
default_entry: 2
remember_last_entry: yes

/+Arch Linux
comment: Any comment
comment: machine-id=PLACEHOLDER_MACHINE

    //linux
        protocol: linux
        path: boot():/vmlinuz-linux
        cmdline: root=UUID=PLACEHOLDER_ROOT rootflags=subvol=@ rw quiet nowatchdog splash
        module_path: boot():/initramfs-linux.img
    
    //linux-lts
        protocol: linux
        path: boot():/vmlinuz-linux-lts
        cmdline: root=UUID=PLACEHOLDER_ROOT rootflags=subvol=@ rw quiet nowatchdog splash
        module_path: boot():/initramfs-linux-lts.img
    
    //linux-zen
        protocol: linux
        path: boot():/vmlinuz-linux-zen
        cmdline: root=UUID=PLACEHOLDER_ROOT rootflags=subvol=@ rw quiet nowatchdog splash
        module_path: boot():/initramfs-linux-zen.img
    
        //Snapshots
        
/+Other systems and bootloaders
    //Windows
        protocol: efi
        path: uuid(PLACEHOLDER_WINDOWS):/EFI/Microsoft/Boot/bootmgfw.efi


# Replace UUID placeholders with actual UUID
sed -i "s/PLACEHOLDER_ROOT/${ROOT_UUID}/g" /boot/limine.conf
sed -i "s/PLACEHOLDER_MACHINE/${MACHINE_UUID}/g" /boot/limine.conf
sed -i "s/PLACEHOLDER_WINDOWS/${WINDOWS_UUID}/g" /boot/limine.conf
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
pacman -S pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber pavucontrol playerctl

# Services are user-level — they start automatically after first login.
```

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
vim /etc/NetworkManager/conf.d/wifi-backend.conf
[device]
wifi.backend=iwd
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
systemctl enable bluetooth
systemctl enable bluetooth.service   
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

#
###########################################################################
###########################################################################
# PART 2 — POST-INSTALLATION
###########################################################################
###########################################################################
#

##  First Boot & AUR Helper (paru)
```bash
# Login as your user, then:
sudo pacman -Syu

# Install paru
pacman -S --needed git base-devel
git clone https://aur.archlinux.org/paru.git /tmp/paru
cd /tmp/paru
makepkg -si
```

---

##  NVIDIA Driver Setup
https://wiki.archlinux.org/title/NVIDIA

```bash
# Find GPU Family name
lspci -k -d ::03xx

# Install NVIDIA open kernel modules (RTX 4070 Ti supports open drivers)
pacman -S nvidia-open-dkms nvidia-utils lib32-nvidia-utils nvidia-settings 
pacman -S libva-nvidia-driver libva-utils
pacman -S vulkan-icd-loader lib32-vulkan-icd-loader

# Prevent nouveau from loading
vim /etc/modprobe.d/blacklist-nouveau.conf
blacklist nouveau
options nouveau modeset=0

# NVIDIA DRM modesetting (required for Wayland)
vim /etc/modprobe.d/nvidia.conf
options nvidia_drm modeset=1 fbdev=1
options nvidia NVreg_UsePageAttributeTable=1
options nvidia NVreg_PreserveVideoMemoryAllocations=1
options nvidia NVreg_EnableGpuFirmware=0

# Enable NVIDIA services for suspend/resume
systemctl enable nvidia-suspend.service
systemctl enable nvidia-hibernate.service
systemctl enable nvidia-resume.service

# Verify modeset
cat /sys/module/nvidia_drm/parameters/modeset   # should be Y
```

---

##  NVIDIA Fine-Tuning (Wayland)
```bash
# Environment variables for Wayland NVIDIA
mkdir -p ~/.config/plasma-workspace/env

vim ~/.config/plasma-workspace/env/nvidia-wayland.sh
export LIBVA_DRIVER_NAME=nvidia
export GBM_BACKEND=nvidia-drm
export __GLX_VENDOR_LIBRARY_NAME=nvidia

# Wayland native apps
export MOZ_ENABLE_WAYLAND=1
export ELECTRON_OZONE_PLATFORM_HINT=auto

# VRR / GSYNC support
export __GL_GSYNC_ALLOWED=1
export __GL_VRR_ALLOWED=1
export __NV_PRIME_RENDER_OFFLOAD=1

# Hardware Acceleration (install nvidia-vaapi-driver!)
export NVD_BACKEND=direct

chmod +x ~/.config/plasma-workspace/env/nvidia-wayland.sh
```

---

## NVIDIA mkinitcpio pacman hook
Update NVIDIA module in initcpio after NVIDIA or kernel update

```bash
mkdir -p /etc/pacman.d/hooks

vim /etc/pacman.d/hooks/nvidia.hook
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
```

---

##  KDE Plasma (Wayland)
```bash
pacman -S plasma-meta kde-applications-meta plasma-login-manager
pacman -S xorg-xwayland xdg-desktop-portal-kde qt6-wayland qt5-wayland
pacman -S papirus-icon-theme
sudo localectl set-x11-keymap pl

# systemctl enable plasma-login-manager
systemctl enable plasmalogin.service
systemctl disable sddm.service

# Check if you have nvidia-drm.modeset=1 in cat /proc/cmdline

reboot
```

---

##  mkinitcpio initramfs HOOKS/MODULES configuration
Configure HOOKS for Btrfs, NVIDIA, and encryption support

```bash
vim /etc/mkinitcpio.conf
MODULES=(btrfs nvidia nvidia_modeset nvidia_uvm nvidia_drm)
BINARIES=()
FILES=()
HOOKS=(base systemd autodetect microcode kms modconf block keyboard sd-vconsole filesystems fsck)
COMPRESSION="zstd"
COMPRESSION_OPTIONS=(-3)

mkinitcpio -P

# Using `systemd` hook set (modern replacement for udev/etc.)
# `kms` hook ensures DRM is loaded early — needed for NVIDIA DRM modesetting.

reboot
```

---

## Verify microcode (amd_ucode)
  https://wiki.archlinux.org/title/Microcode#mkinitcpio
  https://wiki.archlinux.org/title/Microcode#Limine

```bash
lsinitcpio --early /boot/initramfs-linux.img | grep microcode 

# It should print:
#kernel/x86/microcode/
#kernel/x86/microcode/AuthenticAMD.bin
```

---

##  Pacman Configuration
```bash
vim /etc/pacman.conf
Color
VerbosePkgLists
ParallelDownloads = 10
ILoveCandy

# Enable multilib (required for Steam/Wine 32-bit):
# Uncomment [multilib] section:
[multilib]
Include = /etc/pacman.d/mirrorlist
```

---

##  Reflector — Auto Mirror Updates
```bash
pacman -S reflector

vim /etc/xdg/reflector/reflector.conf
--protocol https
--age 12
--latest 10
--sort age
--save /etc/pacman.d/mirrorlist

sudo systemctl enable --now reflector.timer
sudo systemctl enable --now reflector.service
```

---

##  ZRAM Swap
https://wiki.archlinux.org/title/Zram#Using_zram-generator

```bash
pacman -S zram-generator

vim /etc/systemd/zram-generator.conf
[zram0]
zram-size = ram / 2         # 16 GB zram from 32 GB RAM
compression-algorithm = zstd
swap-priority = 100

# Tune swappiness (lower = use RAM more aggressively)
vim /etc/sysctl.d/99-swap.conf
vm.swappiness = 10
vm.vfs_cache_pressure = 50

# Zram enable services
sudo systemctl daemon-reload
sudo systemctl start /dev/zram0
sudo systemctl enable /dev/zram0

reboot

# Verify zram
zramctl
```

---

##  Firewall (UFW)
https://wiki.archlinux.org/title/Uncomplicated_Firewall

```bash
pacman -S ufw

# Default policies
ufw default deny incoming
ufw default allow outgoing

# Allow common services
ufw allow 1900/udp   # UPnP/SSDP (KDE Connect, etc.)
ufw allow 5353/udp   # mDNS
# ufw allow ssh

# Enable
ufw enable
ufw status verbose

# Start and enable "ufw.service" to make it available at boot. 
# Note that this will not work if "iptables.service" is also enabled
systemctl enable --now ufw
```

---

## ZSH SHELL
https://wiki.archlinux.org/title/Zsh

```bash
pacman -S zsh
chsh -s /bin/zsh

# Install OhMyZsh
# https://github.com/ohmyzsh/ohmyzsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# Install plugins zsh-autosuggestions zsh-syntax-highlighting
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting

vim .zshrc
plugins=(
  git
  sudo
  zsh-autosuggestions
  zsh-syntax-highlighting
)

# Install Powerlevel10k
# https://github.com/romkatv/powerlevel10k?tab=readme-ov-file#arch-linux
paru -S --noconfirm zsh-theme-powerlevel10k-git
echo 'source /usr/share/zsh-theme-powerlevel10k/powerlevel10k.zsh-theme' >>~/.zshrc
disable ZSH_THEME in .zshrc
# OR Manually:
# git clone --depth=1 https://github.com/romkatv/powerlevel10k.git "${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k"
# vim .zshrc
# ZSH_THEME="powerlevel10k/powerlevel10k"

# Install MesloLGS font for p10k
# https://www.nerdfonts.com/font-downloads
pacman -S ttf-meslo-nerd
fc-cache -fv

# Configure p10k
p10k configure

# Configure Konsole:
> Konsole → Settings → Edit Current Profile → Appearance
> Select Font: MesloLGS NF Regular
> Select Font size

> General → Enable "Allow bold text"
> Scrolling → Unlimited scrollback

> Settings → Configure Konsole → General
> Select: Remember window size
> Select: Use system monospace font (optional)
```

---

##  TRIM Support (SSD Optimization)
```bash
# Verify TRIM is supported
sudo hdparm -I /dev/sda | grep -i trim

# fstrim timer
sudo systemctl enable fstrim.timer          # SSD TRIM (weekly)
sudo systemctl status fstrim.timer

# Verify discard=async is in fstab (it should be from our mount options)
grep discard /etc/fstab
```

---

##  Snapper — Btrfs Snapshots with Pre/Post System Updates
https://wiki.archlinux.org/title/Snapper#Creating_a_new_configuration

### Install snapper and snap-pac and btrfs-assistant
```bash
pacman -S snapper
pacman -S snap-pac     # automatically creates **pre** and **post** snapshots around every `pacman` transaction
pacman -S btrfs-assistant    # GUI for snapper
```

### Configure snapper for root
```bash
# Create config (snapper will use /.snapshots subvolume we already created)
sudo snapper -c root create-config /

# Optional, if there is a problem because /.snapshots subvolume already exists:
# https://wiki.archlinux.org/title/Snapper#Configuration_of_snapper_and_mount_point
# https://www.reddit.com/r/archlinux/comments/1327vh7/snapper_config_fail/
umount /.snapshots
rm -r /.snapshots
sudo snapper -c root create-config /
btrfs subvolume delete /.snapshots
mkdir /.snapshots
mount -a      # this will use /etc/fstab configuration to mount again /.snapshots subvolume

# Verify
sudo snapper -c root list
sudo cat /etc/snapper/configs/root
```

### Tune snapper cleanup policy
```bash
sudo vim /etc/snapper/configs/root
# Set these values:
TIMELINE_CREATE="no"          # We don't need timeline — snap-pac handles it
NUMBER_CLEANUP="yes"
NUMBER_MIN_AGE="1800"
NUMBER_LIMIT="10"             # Keep last 10 pre/post pairs
NUMBER_LIMIT_IMPORTANT="10"

sudo systemctl enable --now snapper-cleanup.timer

# Optional if timeline disabled:
# sudo systemctl enable --now snapper-timeline.timer 

# Allow user access to snapshots:
sudo snapper -c root set-config ALLOW_USERS=$(whoami)
sudo chmod a+rx /.snapshots
```

---

##  Limine Snapper Sync
Generates boot entries for each snapshot:
https://wiki.archlinux.org/title/Limine#Snapper_snapshot_integration_for_Btrfs

```bash
paru -S limine-snapper-sync

# Enable service — regenerates limine.conf after each snapshot change
#sudo systemctl enable --now limine-snapper-sync.timer
sudo systemctl enable --now limine-snapper-sync.service

# Verify and trigger to check if it works
sudo limine-snapper-sync

# After this, every snapper snapshot will get its own entry in the Limine boot menu.
```

---

## Create limine-mkinitcpio-hook:
Automatically update kernel boot entries in /boot/limine.conf whenever kernels are installed, updated, or removed

```bash
paru -S limine-mkinitcpio-hook

# Commands:
# - limine-install installs Limine to your EFI system partition.
# - limine-install --fallback installs Limine as the default fallback boot loader.
# - limine-update updates Limine and generates an initramfs or UKI depending on your initramfs generator:
# - - For mkinitcpio: run limine-mkinitcpio instead of mkinitcpio
# - limine-scan detects active EFI boot entries (dual boot) and allows you to easily add them to Limine.
# - limine-entry-tool --remove-os "Windows Boot Manager" removes an OS entry matching the specified name, leaving its bootable files intact.

# Some non-compliant UEFI motherboards (e.g., certain MSI boards) have non-standard or broken EFI implementations. They may not work with efibootmgr or kernel-based UEFI detection. To skip UEFI detection and registration and set Limine as the fallback boot loader at the standard EFI path /boot/EFI/BOOT/BOOTX64.EFI, run:
# limine-install --skip-uefi --fallback

# **Note:** The Windows entry uses `boot(windows)` — Limine will scan for the Windows EFI on another disk automatically. Adjust if needed.
```
---

## Create Limine Hook:
Update Limine EFI after limine upgrade

```bash
vim /etc/pacman.d/hooks/99-limine.hook
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

##  Useful Tools
```bash
pacman -S \
  zsh \
  btop \
  fastfetch \
  bat eza fd ripgrep \
  p7zip unrar unzip \
  usbutils \
  print-manager \
  system-config-printer \
  cups \
  cups-pdf \
  cups-filters \
  vlc \
  firefox \
  dolphin \
  kate \
  ark \
  konsole \
  plasma-nm \
  spectacle \
  libreoffice-fresh \
  gimp \
  qbittorrent

pacman -S kde-gtk-config gtk3 gtk4 xdg-utils xdg-user-dirs
reboot
xdg-user-dirs-update    # Create user dirs like Home, Documents etc

pacman -S \
  jdk21-openjdk \
  nodejs \
  npm \
  dbeaver \
  pycharm-community-edition \
  idea-community-edition \
  python \
  python-pip \
  uv \
  podman \
  podman-compose \
  podman-docker
  
paru -S brave-bin visual-studio-code-bin

## Sublime text
curl -O https://download.sublimetext.com/sublimehq-pub.gpg && sudo pacman-key --add sublimehq-pub.gpg && sudo pacman-key --lsign-key 8A8F901A && rm sublimehq-pub.gpg
echo -e "\n[sublime-text]\nServer = https://download.sublimetext.com/arch/stable/x86_64" | sudo tee -a /etc/pacman.conf
sudo pacman -Syu sublime-text

## Printer - Brother
# Driver:
paru -S brother-hl1210w

# KDE -> System Settings -> Printers -> Add New...
# Connection:
lpd://192.168.2.140/
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
  vulkan-tools \
  nvtop                # GPU monitoring

# Enable GameMode daemon
systemctl --user enable gamemoded   # Run after first login

# In Steam: Right-click game → Properties → Launch Options: `gamemoderun %command%`
# For Proton: Enable in Steam → Settings → Compatibility → Enable Steam Play for all games → Use Proton Experimental
```

---

##  Optional: AMD CPU Tuning (Ryzen 7800X3D)
https://wiki.archlinux.org/title/KDE#Power_management

```bash
# Install CPU power management
pacman -S power-profiles-daemon powerdevil

# Limit peak power slightly (optional — 7800X3D runs hot under load)
# Ryzen power plans via KDE Power Management (power-profiles-daemon)
sudo systemctl enable --now power-profiles-daemon

# Install CoreCtrl for fine-grained AMD control (AUR)
paru -S corectrl
```

---

##  Optional: Flatpak
```bash
pacman -S flatpak
# Flatpak remote
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

# Flatpak integration
pacman -S flatpak-kcm
```

---

##  Optional: Firmware updates tool
```bash
# Install firmware updates tool:
pacman -S fwupd
reboot
# Commands after: 
# fwupdmgr get-devices, 
# fwupdmgr get-updates
```

---

##  Optional: Security Hardening
```bash
# Kernel hardening sysctl
vim /etc/sysctl.d/99-security.conf
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


sudo sysctl --system
```

---

##  System Update Workflow
With `snap-pac` installed, every `pacman` or `paru` update automatically creates pre/post snapshots.

```bash
---
##################################
# Maintenance
##################################

# Full system update (snapshots happen automatically)
sudo pacman -Syu

# Or with paru (also covers AUR)
paru -Syu

# List packages installed by you (from pacman)
pacman -Qen

# List packages installed by you (except pacman, eg from paru)
pacman -Qem

# Export packages to txt file
pacman -Qeq > my_packages.txt

# Remove package, not used dependencies and config files
sudo pacman -Rns xxx

# Check orphaned packages
sudo pacman -Qdt

# Cleanup orphaned packages after removal
sudo pacman -Qtdq | sudo pacman -Rns -
# OR:
sudo pacman -Rns $(pacman -Qtdq)

# Cleanup pacman cache (required package: pacman-contrib)
sudo paccache -r

---
##################################
# Packages
##################################

# Export packages to txt
pacman -Qqen > pkglist-official.txt
pacman -Qqem > pkglist-aur.txt

# Import packages from txt
sudo pacman -S --needed - < pkglist-official.txt
paru -S --needed - < pkglist-aur.txt

---
##################################
# Backups
##################################

# View snapshots
sudo snapper -c root list

# Manual snapshot (e.g., before risky change)
sudo snapper -c root create -d "Before my experiment" --type single

# Rollback if something breaks
sudo snapper -c root rollback 5   # Rollback to snapshot #5
reboot

---
##################################
# Logs and Errors
##################################

# Check Ethernet info
lspci -k | grep -A 3 Ethernet
inxi -N
ip link
ip link show enp11s0

# Check failed services
systemctl --failed

# Check for logs from last boot
journalctl -b
journalctl -b -p err
journalctl -b -1

# Problems from last boot
journalctl -p 3 -xb

# Kernel problems
journalctl -k

sudo dmesg -T --level=err,warn
sudo dmesg | less


```
