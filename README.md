# Arch Linux Installation Guide (2026)
**Hardware:** MSI MAG B650 Tomahawk WiFi, Ryzen 7800X3D, NVIDIA RTX 4070 Ti, 32GB RAM  
**Target Disk:** /dev/sda (separate SSD)  
**Features:** Btrfs with snapshots, GRUB dual boot, KDE Plasma, gaming-ready, NVIDIA optimized

---

## SECTION 1: INSTALLATION

### 1.1 Boot into Arch Installation Medium

1. **Boot from Arch ISO** in UEFI mode (not legacy BIOS)
   - Select "Arch Linux install medium (x86_64, UEFI)"

2. **Set larger console font** (optional, for better readability):
   ```bash
   setfont ter-132b
   ```

3. **Verify boot mode** (should show UEFI):
   ```bash
   cat /sys/firmware/efi/fw_platform_size
   # Should output: 64
   ```

4. **Connect to internet**:
   ```bash
   # For wired connection, it should work automatically
   ping -c 3 archlinux.org
   
   # For WiFi (if needed):
   iwctl
   # device list
   # station wlan0 scan
   # station wlan0 get-networks
   # station wlan0 connect "YOUR_SSID"
   # exit
   ```

5. **Update system clock**:
   ```bash
   timedatectl set-ntp true
   timedatectl status
   ```

---

### 1.2 Disk Partitioning (Btrfs Best Practices)

**IMPORTANT:** We'll create a layout that allows easy snapshot restoration and keeps /home separate.

1. **Identify your disk** (should be /dev/sda):
   ```bash
   lsblk
   fdisk -l
   ```

2. **Partition the disk** using `gdisk`:
   ```bash
   gdisk /dev/sda
   ```

   Create the following partitions:
   
   - **EFI System Partition** (ESP):
     - Command: `n` (new partition)
     - Partition number: 1
     - First sector: (press Enter for default)
     - Last sector: `+1G` (1GB is recommended for dual boot)
     - Hex code: `ef00` (EFI System)
   
   - **Root Partition**:
     - Command: `n`
     - Partition number: 2
     - First sector: (press Enter)
     - Last sector: (press Enter to use remaining space)
     - Hex code: `8300` (Linux filesystem)
   
   - Write changes: `w` and confirm

3. **Verify partitions**:
   ```bash
   lsblk /dev/sda
   # Should show:
   # sda1 (1G) - EFI
   # sda2 (rest) - Linux filesystem
   ```

---

### 1.3 Format Partitions

1. **Format EFI partition**:
   ```bash
   mkfs.fat -F32 /dev/sda1
   ```

2. **Format root partition with Btrfs**:
   ```bash
   mkfs.btrfs -L ArchRoot /dev/sda2
   ```

---

### 1.4 Create Btrfs Subvolumes (Critical for Snapshots!)

This layout is optimized for Snapper snapshots and system rollbacks:

1. **Mount the Btrfs filesystem**:
   ```bash
   mount /dev/sda2 /mnt
   ```

2. **Create subvolumes**:
   ```bash
   # Root subvolume (will be snapshotted)
   btrfs subvolume create /mnt/@
   
   # Home subvolume (separate, won't be affected by root snapshots)
   btrfs subvolume create /mnt/@home
   
   # Snapshots storage (outside of @ to preserve snapshots during rollback)
   btrfs subvolume create /mnt/@snapshots
   
   # Variable data (logs, cache - exclude from snapshots)
   btrfs subvolume create /mnt/@var_log
   
   # Package cache (exclude from snapshots to save space)
   btrfs subvolume create /mnt/@var_cache_pacman
   
   # Docker (if you'll use containers)
   btrfs subvolume create /mnt/@var_lib_docker
   
   # LibVirt (if you'll use VMs)
   btrfs subvolume create /mnt/@var_lib_libvirt
   ```

3. **Unmount**:
   ```bash
   umount /mnt
   ```

---

### 1.5 Mount Subvolumes with Optimal Options

**Btrfs mount options explained:**
- `compress=zstd:1` - Transparent compression (level 1 for speed/compression balance)
- `noatime` - Don't update access times (improves performance)
- `space_cache=v2` - Improved free space cache
- `discard=async` - Async TRIM for SSD
- `ssd` - SSD optimizations

1. **Mount root subvolume**:
   ```bash
   mount -o compress=zstd:1,noatime,space_cache=v2,discard=async,ssd,subvol=@ /dev/sda2 /mnt
   ```

2. **Create mount points**:
   ```bash
   mkdir -p /mnt/{boot,home,.snapshots,var/log,var/cache/pacman,var/lib/docker,var/lib/libvirt}
   ```

3. **Mount EFI partition**:
   ```bash
   mount /dev/sda1 /mnt/boot
   ```

4. **Mount other subvolumes**:
   ```bash
   mount -o compress=zstd:1,noatime,space_cache=v2,discard=async,ssd,subvol=@home /dev/sda2 /mnt/home
   mount -o compress=zstd:1,noatime,space_cache=v2,discard=async,ssd,subvol=@snapshots /dev/sda2 /mnt/.snapshots
   mount -o compress=zstd:1,noatime,space_cache=v2,discard=async,ssd,subvol=@var_log /dev/sda2 /mnt/var/log
   mount -o compress=zstd:1,noatime,space_cache=v2,discard=async,ssd,subvol=@var_cache_pacman /dev/sda2 /mnt/var/cache/pacman
   mount -o compress=zstd:1,noatime,space_cache=v2,discard=async,ssd,subvol=@var_lib_docker /dev/sda2 /mnt/var/lib/docker
   mount -o compress=zstd:1,noatime,space_cache=v2,discard=async,ssd,subvol=@var_lib_libvirt /dev/sda2 /mnt/var/lib/libvirt
   ```

5. **Verify mounts**:
   ```bash
   lsblk
   mount | grep /mnt
   ```

---

### 1.6 Install Base System

1. **Install essential packages**:
   ```bash
   pacstrap -K /mnt base base-devel linux linux-headers \
     linux-zen linux-zen-headers linux-firmware\
     btrfs-progs amd-ucode nano vim networkmanager sudo git \
     bash-completion man-db man-pages
   ```

   **Package explanations:**
   - `base`, `base-devel` - Essential system and development tools
   - `linux`, `linux-firmware`, `linux-headers` - Kernel and firmware
   - `btrfs-progs` - Btrfs utilities
   - `amd-ucode` - AMD CPU microcode (for Ryzen 7800X3D)
   - `networkmanager` - Network management
   - Rest - Basic utilities

---

### 1.7 Configure the System

1. **Generate fstab**:
   ```bash
   genfstab -U /mnt >> /mnt/etc/fstab
   ```

2. **Verify fstab** (important - check subvolume mounts):
   ```bash
   cat /mnt/etc/fstab
   ```

3. **Chroot into the system**:
   ```bash
   arch-chroot /mnt
   ```

4. **Set timezone**:
   ```bash
   ln -sf /usr/share/zoneinfo/Europe/Warsaw /etc/localtime
   hwclock --systohc
   ```

5. **Configure localization**:
   ```bash
   # Edit /etc/locale.gen and uncomment:
   # en_US.UTF-8 UTF-8
   # pl_PL.UTF-8 UTF-8 (if you want Polish)
   
   nano /etc/locale.gen
   # Uncomment: en_US.UTF-8 UTF-8
   
   locale-gen
   
   echo "LANG=en_US.UTF-8" > /etc/locale.conf
   echo "KEYMAP=pl" > /etc/vconsole.conf
   ```

6. **Set hostname**:
   ```bash
   echo "archgaming" > /etc/hostname
   ```

7. **Configure hosts file**:
   ```bash
   cat > /etc/hosts << EOF
   127.0.0.1   localhost
   ::1         localhost
   127.0.1.1   archgaming.localdomain archgaming
   EOF
   ```

8. **Set root password**:
   ```bash
   passwd
   ```

9. **Create user account**:
   ```bash
   useradd -m -G wheel,audio,video,optical,storage -s /bin/bash yourusername
   passwd yourusername
   ```

10. **Configure sudo**:
    ```bash
    EDITOR=nano visudo
    # Uncomment this line:
    # %wheel ALL=(ALL:ALL) ALL
    ```

11. **Enable NetworkManager**:
    ```bash
    systemctl enable NetworkManager
    ```

---

### 1.8 Install and Configure GRUB (Dual Boot)

1. **Install GRUB and tools**:
   ```bash
   pacman -S grub efibootmgr os-prober dosfstools mtools
   ```

2. **Enable os-prober** (to detect Windows):
   ```bash
   echo "GRUB_DISABLE_OS_PROBER=false" >> /etc/default/grub
   ```
   Add for default boot last saved:
      GRUB_DEFAULT=saved
      GRUB_SAVEDEFAULT=true

3. **Mount Windows EFI partition** (if Windows is on a different disk):
   ```bash
   # Find Windows EFI partition (usually on another disk)
   lsblk -f
   
   # Mount it temporarily (replace sdX1 with actual partition)
   mkdir -p /mnt/windows-efi
   mount /dev/sdX1 /mnt/windows-efi
   ```

4. **Install GRUB to EFI**:
   ```bash
   grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
   ```

5. **Generate GRUB configuration**:
   ```bash
   grub-mkconfig -o /boot/grub/grub.cfg
   ```
   
   **Note:** You should see "Found Windows Boot Manager" if Windows was detected.

6. **Unmount Windows EFI** (if mounted):
   ```bash
   umount /mnt/windows-efi
   ```

---

### 1.9 Configure mkinitcpio for Btrfs

1. **Edit mkinitcpio.conf**:
   ```bash
   nano /etc/mkinitcpio.conf
   ```

2. **Add btrfs to MODULES** (we'll add NVIDIA modules later in post-installation):
   ```
   MODULES=(btrfs)
   ```

3. **Regenerate initramfs**:
   ```bash
   mkinitcpio -P
   ```

---

### 1.10 Final Steps Before Reboot

1. **Exit chroot**:
   ```bash
   exit
   ```

2. **Unmount all partitions**:
   ```bash
   umount -R /mnt
   ```

3. **Reboot**:
   ```bash
   reboot
   ```

4. **Remove installation media** and boot into your new system

5. **Login** with your user account

---

## SECTION 2: POST-INSTALLATION

### 2.1 Network Configuration

1. **Connect to WiFi** (if needed):
   ```bash
   nmcli device wifi list
   nmcli device wifi connect "YOUR_SSID" password "YOUR_PASSWORD"
   ```

2. **Verify internet**:
   ```bash
   ping -c 3 archlinux.org
   ```

---

### 2.2 Install AUR Helper (Paru)

1. **Install paru**:
   ```bash
   cd /tmp
   git clone https://aur.archlinux.org/paru.git
   cd paru
   makepkg -si
   ```

2. **Test paru**:
   ```bash
   paru --version
   ```

---

### 2.3 Install and Configure NVIDIA Drivers

**IMPORTANT:** For RTX 4070 Ti (Ada Lovelace architecture), use nvidia-dkms for best compatibility.

1. **Enable multilib repository** (for 32-bit gaming libraries):
   ```bash
   sudo nano /etc/pacman.conf
   ```
   
   Uncomment these lines:
   ```
   [multilib]
   Include = /etc/pacman.d/mirrorlist
   ```

2. **Update package database**:
   ```bash
   sudo pacman -Syu
   ```

3. **Install NVIDIA drivers**:
   ```bash
   sudo pacman -S nvidia-dkms nvidia-utils lib32-nvidia-utils nvidia-settings
   ```

4. **Configure mkinitcpio for NVIDIA**:
   ```bash
   sudo nano /etc/mkinitcpio.conf
   ```
   
   Update MODULES line:
   ```
   MODULES=(btrfs nvidia nvidia_modeset nvidia_uvm nvidia_drm)
   ```
   
   **Remove 'kms' from HOOKS** (if present):
   ```
   # Change from: HOOKS=(base udev autodetect kms modconf block filesystems keyboard fsck)
   # To:          HOOKS=(base udev autodetect modconf block filesystems keyboard fsck)
   ```

5. **Configure NVIDIA kernel parameters**:
   ```bash
   sudo nano /etc/default/grub
   ```
   
   Modify GRUB_CMDLINE_LINUX_DEFAULT:
   ```
   GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet nvidia-drm.modeset=1 nvidia-drm.fbdev=1"
   ```

6. **Create NVIDIA pacman hook** (auto-rebuilds initramfs on driver updates):
   ```bash
   sudo mkdir -p /etc/pacman.d/hooks
   sudo nano /etc/pacman.d/hooks/nvidia.hook
   ```
   
   Add this content:
   ```
   [Trigger]
   Operation=Install
   Operation=Upgrade
   Operation=Remove
   Type=Package
   Target=nvidia-dkms
   Target=linux
   
   [Action]
   Description=Updating NVIDIA module in initcpio
   Depends=mkinitcpio
   When=PostTransaction
   NeedsTargets
   Exec=/bin/sh -c 'while read -r trg; do case $trg in linux) exit 0; esac; done; /usr/bin/mkinitcpio -P'
   ```

7. **Regenerate initramfs and GRUB config**:
   ```bash
   sudo mkinitcpio -P
   sudo grub-mkconfig -o /boot/grub/grub.cfg
   ```

8. **Reboot to load NVIDIA drivers**:
   ```bash
   sudo reboot
   ```

9. **Verify NVIDIA is working** (after reboot):
   ```bash
   nvidia-smi
   lsmod | grep nvidia
   ```

---

### 2.4 Configure Snapshots with Snapper

1. **Install snapshot tools**:
   ```bash
   paru -S snapper-support btrfs-assistant inotify-tools
   ```
   
   **What gets installed:**
   - `snapper` - Snapshot management
   - `snap-pac` - Automatic pre/post snapshots on pacman operations
   - `grub-btrfs` - Add snapshots to GRUB boot menu
   - `btrfs-assistant` - GUI for managing snapshots
   - `inotify-tools` - Automatic GRUB update when snapshots change

2. **Configure Snapper for root**:
   ```bash
   # Unmount .snapshots (snapper will recreate it)
   sudo umount /.snapshots
   sudo rm -rf /.snapshots
   
   # Create snapper config for root
   sudo snapper -c root create-config /
   
   # Delete the .snapshots subvolume created by snapper
   sudo btrfs subvolume delete /.snapshots
   
   # Recreate mount point and remount our @snapshots subvolume
   sudo mkdir /.snapshots
   sudo mount -o compress=zstd:1,noatime,space_cache=v2,discard=async,ssd,subvol=@snapshots /dev/sda2 /.snapshots
   
   # Set correct permissions
   sudo chmod 750 /.snapshots
   sudo chown :wheel /.snapshots
   ```

3. **Configure snapper settings**:
   ```bash
   sudo nano /etc/snapper/configs/root
   ```
   
   Modify these values:
   ```
   ALLOW_GROUPS="wheel"
   TIMELINE_CREATE="yes"
   TIMELINE_CLEANUP="yes"
   
   # Snapshot retention (adjust to your needs)
   TIMELINE_LIMIT_HOURLY="5"
   TIMELINE_LIMIT_DAILY="7"
   TIMELINE_LIMIT_WEEKLY="4"
   TIMELINE_LIMIT_MONTHLY="6"
   TIMELINE_LIMIT_YEARLY="0"
   ```

4. **Enable snapper services**:
   ```bash
   sudo systemctl enable --now snapper-timeline.timer
   sudo systemctl enable --now snapper-cleanup.timer
   sudo systemctl enable --now grub-btrfsd
   ```

5. **Configure btrfs-assistant** (for proper snapshot mapping):
   ```bash
   sudo nano /etc/btrfs-assistant.conf
   ```
   
   Add at the end under `[Subvol-Mapping]`:
   ```
   root = "@.snapshots,@,UUID_OF_SDA2"
   ```
   
   Get UUID with:
   ```bash
   lsblk -f | grep sda2
   # Or from /etc/fstab
   ```

6. **Create initial snapshot**:
   ```bash
   sudo snapper -c root create --description "Fresh installation"
   ```

7. **Update GRUB to include snapshots**:
   ```bash
   sudo grub-mkconfig -o /boot/grub/grub.cfg
   ```

8. **Enable grub-btrfs overlayfs hook** (allows booting snapshots in read-write mode):
   ```bash
   sudo nano /etc/mkinitcpio.conf
   ```
   
   Add `grub-btrfs-overlayfs` to the end of HOOKS:
   ```
   HOOKS=(base udev autodetect modconf block filesystems keyboard fsck grub-btrfs-overlayfs)
   ```
   
   Regenerate initramfs:
   ```bash
   sudo mkinitcpio -P
   ```

---

### 2.5 Configure zram (Swap)

1. **Install zram-generator**:
   ```bash
   sudo pacman -S zram-generator
   ```

2. **Configure zram**:
   ```bash
   sudo nano /etc/systemd/zram-generator.conf
   ```
   
   Add:
   ```
   [zram0]
   zram-size = ram / 2
   compression-algorithm = zstd
   ```

3. **Start zram**:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl start systemd-zram-setup@zram0.service
   ```

4. **Verify zram**:
   ```bash
   lsblk
   swapon --show
   ```

---

### 2.6 Enable TRIM Support for SSD

1. **Enable periodic TRIM**:
   ```bash
   sudo systemctl enable fstrim.timer
   ```

2. **Verify TRIM support**:
   ```bash
   lsblk --discard
   # Non-zero values in DISC-GRAN and DISC-MAX indicate TRIM support
   ```

---

### 2.7 Install KDE Plasma

1. **Install Plasma and applications**:
   ```bash
   sudo pacman -S plasma-meta kde-applications-meta \
     plasma-wayland-session xorg-xwayland \
     sddm sddm-kcm
   ```

2. **Enable SDDM** (display manager):
   ```bash
   sudo systemctl enable sddm
   ```

3. **For NVIDIA + Wayland** (configure SDDM):
   ```bash
   sudo nano /etc/sddm.conf.d/10-wayland.conf
   ```
   
   Add:
   ```
   [General]
   DisplayServer=wayland
   GreeterEnvironment=QT_WAYLAND_SHELL_INTEGRATION=layer-shell
   
   [Wayland]
   CompositorCommand=kwin_wayland --no-global-shortcuts
   ```

---

### 2.8 Install Gaming Essentials

1. **Install Steam and gaming tools**:
   ```bash
   sudo pacman -S steam wine-staging winetricks gamemode lib32-gamemode \
     mangohud lib32-mangohud goverlay lutris \
     vulkan-icd-loader lib32-vulkan-icd-loader
   ```

2. **Install ProtonUp-Qt** (easy Proton-GE installation):
   ```bash
   paru -S protonup-qt
   ```

3. **Enable gamemode for your user**:
   ```bash
   sudo usermod -aG gamemode $USER
   ```

---

### 2.9 Configure Firewall (UFW)

1. **Install and configure UFW**:
   ```bash
   sudo pacman -S ufw
   ```

2. **Configure basic rules**:
   ```bash
   sudo ufw default deny incoming
   sudo ufw default allow outgoing
   sudo ufw allow from 192.168.0.0/24  # Adjust to your local network
   ```

3. **Enable UFW**:
   ```bash
   sudo systemctl enable --now ufw
   sudo ufw enable
   ```

4. **Check status**:
   ```bash
   sudo ufw status verbose
   ```

---

### 2.10 Audio, Bluetooth, and Additional Services

1. **Audio is already configured** (PipeWire comes with KDE Plasma)

2. **Enable Bluetooth**:
   ```bash
   sudo systemctl enable --now bluetooth
   ```

3. **Install additional codecs**:
   ```bash
   sudo pacman -S pipewire-jack lib32-pipewire-jack \
     gst-plugins-good gst-plugins-bad gst-plugins-ugly \
     gst-libav ffmpeg
   ```

---

### 2.11 Performance Tweaks for Ryzen 7800X3D

1. **Install AMD-specific tools**:
   ```bash
   sudo pacman -S ryzen_smu zenpower3-dkms
   ```

2. **Enable full AMD P-State driver**:
   ```bash
   sudo nano /etc/default/grub
   ```
   
   Add to GRUB_CMDLINE_LINUX_DEFAULT:
   ```
   amd_pstate=active
   ```
   
   Update GRUB:
   ```bash
   sudo grub-mkconfig -o /boot/grub/grub.cfg
   ```

3. **Install CPU frequency tools**:
   ```bash
   sudo pacman -S cpupower
   sudo systemctl enable --now cpupower
   ```

---

### 2.12 Final System Configuration

1. **Update system**:
   ```bash
   sudo pacman -Syu
   ```

2. **Install additional useful tools**:
   ```bash
   sudo pacman -S htop btop neofetch fastfetch \
     firefox chromium vlc gimp inkscape libreoffice-fresh \
     discord telegram-desktop qbittorrent
   ```

3. **Configure automatic mirrorlist updates**:
   ```bash
   sudo pacman -S reflector
   sudo systemctl enable --now reflector.timer
   ```

4. **Set up automatic snapshot before system updates**:
   ```bash
   # This is already configured by snap-pac!
   # Verify it's working:
   sudo pacman -S neofetch
   # Check if snapshot was created:
   sudo snapper -c root list
   ```

5. **Reboot into KDE Plasma**:
   ```bash
   sudo reboot
   ```

---

## TESTING SNAPSHOT ROLLBACK

After booting into KDE, test the snapshot system:

1. **Install a test package**:
   ```bash
   sudo pacman -S cowsay
   ```

2. **Verify snapshots were created**:
   ```bash
   sudo snapper -c root list
   # You should see pre/post snapshots
   ```

3. **Reboot and access GRUB**:
   ```bash
   sudo reboot
   ```

4. **In GRUB menu:**
   - Select "Arch Linux snapshots"
   - Choose a snapshot from before cowsay installation
   - Boot into it

5. **Verify cowsay is gone**:
   ```bash
   cowsay "test"  # Should fail: command not found
   ```

6. **To permanently rollback to a snapshot**:
   ```bash
   # Boot into the desired snapshot from GRUB
   # Then run:
   sudo snapper -c root rollback <snapshot_number>
   sudo reboot
   ```

---

## DUAL BOOT NOTES

### If Windows doesn't appear in GRUB:

1. **Mount Windows EFI partition**:
   ```bash
   # Find Windows EFI (usually on different disk)
   lsblk -f
   sudo mkdir -p /mnt/win-efi
   sudo mount /dev/sdX1 /mnt/win-efi  # Replace sdX1
   ```

2. **Run os-prober and update GRUB**:
   ```bash
   sudo os-prober
   sudo grub-mkconfig -o /boot/grub/grub.cfg
   ```

3. **Unmount**:
   ```bash
   sudo umount /mnt/win-efi
   ```

### Alternative: Use UEFI boot menu
- Press F11/F12 during boot to select Windows directly from UEFI

---

## MAINTENANCE COMMANDS

### Snapshot Management:
```bash
# List snapshots
sudo snapper -c root list

# Create manual snapshot
sudo snapper -c root create --description "Before major change"

# Delete snapshot
sudo snapper -c root delete <number>

# Compare snapshots
sudo snapper -c root status <num1>..<num2>

# GUI management
btrfs-assistant
```

### System Updates:
```bash
# Full system update (snapshots auto-created by snap-pac)
sudo pacman -Syu

# Update AUR packages
paru -Syu

# Clean package cache
sudo pacman -Sc
```

### Check System Health:
```bash
# Check Btrfs filesystem
sudo btrfs scrub start /
sudo btrfs scrub status /

# Check NVIDIA
nvidia-smi

# Check services
sudo systemctl --failed
```

---

## TROUBLESHOOTING

### NVIDIA not loading:
```bash
# Check if modules are loaded
lsmod | grep nvidia

# Rebuild initramfs
sudo mkinitcpio -P

# Check kernel parameters
cat /proc/cmdline
```

### GRUB not showing snapshots:
```bash
# Restart grub-btrfsd
sudo systemctl restart grub-btrfsd

# Manually update GRUB
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

### Can't boot after snapshot rollback:
```bash
# Boot from live USB
# Mount your system
mount -o subvol=@ /dev/sda2 /mnt
mount /dev/sda1 /mnt/boot
arch-chroot /mnt

# Reinstall GRUB
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
mkinitcpio -P
```

---

## ADDITIONAL RESOURCES

- **ArchWiki:** https://wiki.archlinux.org
- **Snapper Documentation:** https://wiki.archlinux.org/title/Snapper
- **Btrfs Guide:** https://wiki.archlinux.org/title/Btrfs
- **NVIDIA Guide:** https://wiki.archlinux.org/title/NVIDIA
- **Gaming Guide:** https://wiki.archlinux.org/title/Gaming

---
