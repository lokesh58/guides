# Arch Linux Installation Guide

[Arch Wiki](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LUKS_on_a_partition_with_TPM2_and_Secure_Boot) being followed as a reference for this guide.

## Prerequisites

It is assumed that bootable Arch Linux media is created and system is booted from it.

## Minimal Installation

### Format Disk

This guide will use the following partitions:

1. EFI (512 MiB) -> type = EFI
2. LUKS Root (Rest) -> type = Linux Root

You can use your favorite tool to create these partitions. The guide doesn't cover that part, and official Arch Wiki has a good guide on that.
**Note:** This guide assumes that the disk is `/dev/nvme0n1` and partitions are `nvme0n1p1` and `nvme0n1p2`.
ZRAM will be enabled later in the guide, hence a swap partition is not created.

```bash
mkfs.fat -F32 -n EFI /dev/nvme0n1p1
cryptsetup luksFormat /dev/nvme0n1p2 # keep empty passphrase as it will be removed later
cryptsetup open /dev/nvme0n1p2 root
mkfs.btrfs -L LINUX_ROOT /dev/mapper/root
```

### Create BTRFS Sub-volumes

We will create a flat sub-volume structure. Following the archinstall script recommended structure.

**Note:** We are mounting to `/mnt` and later unmounting it because creating btrfs subvolumes requires the filesystem to be mounted.

```bash
mount /dev/mapper/root /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@log
btrfs subvolume create /mnt/@pkg
btrfs subvolume create /mnt/@.snapshots
umount /mnt
```

### Mount all the partitions and sub-volumes

All mount options with their defaults can be found in [wiki](https://man.archlinux.org/man/btrfs.5.en).
Explanation of mount options:

- `noatime`: Don't update access time on files (default is `relatime`, which can cause performance issues)
- `compress=zstd`: Compress data using zstd (default is none)
- `subvol=<subvolume>`: Mount the specified subvolume
- `umask=0077`: Set the umask to 0077 (default is 0022, which is too permissive for EFI partitions)

```bash
mount -o noatime,compress=zstd,subvol=@ /dev/mapper/root /mnt
mkdir -p /mnt/home /mnt/var/log /mnt/var/cache/pacman/pkg /mnt/.snapshots /mnt/efi
mount -o noatime,compress=zstd,subvol=@home /dev/mapper/root /mnt/home
mount -o noatime,compress=zstd,subvol=@log /dev/mapper/root /mnt/var/log
mount -o noatime,compress=zstd,subvol=@pkg /dev/mapper/root /mnt/var/cache/pacman/pkg
mount -o noatime,compress=zstd,subvol=@.snapshots /dev/mapper/root /mnt/.snapshots
mount -o umask=0077 /dev/nvme0n1p1 /mnt/efi
```

### Install Base System

First update the pacman mirrorlist to use the fastest mirror.

```bash
reflector --country India --age 12 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
```

Before installing packages, enable the multilib repository which is needed for 32-bit packages:

```bash
# Edit pacman.conf to enable multilib
vim /etc/pacman.conf
```

Add/uncomment the following line to the file:

```conf
[multilib]
Include = /etc/pacman.d/mirrorlist
```

Now, install the base system.

**Note:**

1. `intel-ucode` is used for Intel processors. If you are using AMD, use `amd-ucode` instead.
2. Use `nano` instead of `neovim` if you are not familiar with it.

Below is a description of each package being installed:

**Base System:**

- `base`: Minimal package set for a basic Arch Linux installation
- `linux`: The Linux kernel and modules
- `linux-firmware`: Firmware files for Linux
- `intel-ucode`: Microcode updates for Intel CPUs
- `btrfs-progs`: BTRFS filesystem utilities

**Audio:**

- `sof-firmware`: Firmware for Sound Open Framework (required for modern Intel audio)
- `alsa-utils`: Advanced Linux Sound Architecture utilities (provides tools like alsamixer)
- `pipewire`: Modern audio/video server
- `lib32-pipewire`: 32-bit support for PipeWire
- `wireplumber`: Session/policy manager for PipeWire
- `pipewire-alsa`: ALSA client support for PipeWire
- `pipewire-pulse`: PulseAudio client support for PipeWire
- `pipewire-jack`: JACK client support for PipeWire
- `lib32-pipewire-jack`: 32-bit JACK client support

**System Management:**

- `networkmanager`: Network connection manager
- `neovim`: Advanced text editor (can be replaced with nano)
- `man-db`: Manual page reader
- `man-pages`: Linux manual pages
- `texinfo`: GNU documentation system
- `sudo`: Privilege elevation tool
- `reflector`: Pacman mirror list manager

**Security:**

- `apparmor`: Mandatory Access Control (MAC) system
- `sbctl`: Secure Boot key manager

**Bluetooth:**

- `bluez`: Bluetooth protocol stack
- `bluez-utils`: Command line utilities for Bluetooth
- `pipewire-audio`: PipeWire Bluetooth audio support

**Optional Packages (Can be installed later if needed):**

**Audio Enhancements:**

- `pipewire-audio`: Additional Bluetooth audio support for PipeWire
- `pipewire-zeroconf`: Network audio support via Avahi
- `pipewire-x11-bell`: X11 bell support
- `pipewire-v4l2`: Video4Linux2 support
- `easyeffects`: Audio effects and equalizer
- `helvum`: Patchbay for audio routing
- `pavucontrol`: PulseAudio volume control GUI
- `realtime-privileges`: Realtime privileges for better audio performance
- `rtkit`: Realtime scheduling for PipeWire

**Bluetooth:**

- `bluez`: Bluetooth support
- `bluez-utils`: Bluetooth utilities
- `blueman`: Bluetooth manager (GUI)

**Audio Codecs:**

- `opus`: High-quality audio codec
- `sbc`: Low-complexity audio codec
- `libfdk-aac`: AAC codec support
- `lame`: MP3 encoding support
- `flac`: Free Lossless Audio Codec

**Development:**

- `pipewire-docs`: Documentation for PipeWire
- `pipewire-jack-client`: Use PipeWire as a JACK client
- `gst-plugin-pipewire`: GStreamer plugin for PipeWire

**Professional Audio:**

- `ardour`: Digital Audio Workstation
- `carla`: Audio plugin host
- `reaper`: Digital Audio Workstation (proprietary)
- `pipewire-ffado`: FireWire audio device support

```bash
pacstrap -K /mnt base linux linux-firmware intel-ucode btrfs-progs sof-firmware alsa-utils pipewire lib32-pipewire wireplumber pipewire-alsa pipewire-pulse pipewire-jack lib32-pipewire-jack bluez bluez-utils pipewire-audio networkmanager neovim man-db man-pages texinfo sudo reflector apparmor sbctl
```

Generate fstab so that the system can mount the partitions automatically at boot.

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

### Base Configurations

Enter the installed system:

```bash
arch-chroot /mnt
```

#### pacman Configuration

Edit the pacman.conf to enable multilib:

```conf
...
[multilib]
Include = /etc/pacman.d/mirrorlist
...
```

Configure reflector:

```bash
nvim /etc/xdg/reflector/reflector.conf
```

```conf
--save /etc/pacman.d/mirrorlist
--country India
--protocol https
--age 12
--sort rate
```

Enable reflector timer to automatically update the mirrorlist from time to time:

```bash
systemctl enable reflector.timer
```

Optionally, setup auto cleanup of pacman cache:

```bash
pacman -S pacman-contrib
systemctl enable paccache.timer
```

#### Date and Time

Set your local timezone, and enable systemd-timesyncd.service to automatically update the date and time:

```bash
ln -sf /usr/share/zoneinfo/Asia/Kolkata /etc/localtime
hwclock --systohc
systemctl enable systemd-timesyncd.service
```

#### Localization

Generate locale:

```bash
nvim /etc/locale.gen
```

Uncomment the line for your locale:

```conf
en_IN UTF-8
```

Generate locale:

```bash
locale-gen
```

Set your locale and keymap:

```bash
echo "LANG=en_IN" >> /etc/locale.conf
echo "KEYMAP=us" >> /etc/vconsole.conf
```

#### Network Configuration

Set your hostname:

```bash
echo "aoitrur" >> /etc/hostname
```

Edit the hosts file:

```bash
nvim /etc/hosts
```

```conf
127.0.0.1       localhost
127.0.0.1       localhost
127.0.1.1       aoitrur
```

Enable NetworkManager and mask systemd-networkd to prevent conflicts:

```bash
systemctl enable NetworkManager
systemctl mask systemd-networkd
```

#### Add a non-root user

Add a non-root user to wheel group, and enable sudo access for wheel group:

```bash
useradd -mG wheel lokesh58
passwd lokesh58
EDITOR=nvim visudo # uncomment %wheel ALL=(ALL) ALL
```

### Setup Boot

#### Setup Unified Kernel Image

Create a directory for `cmdline.d`, used to specify kernel parameters:

```bash
mkdir /etc/cmdline.d
```

Create a file for kernel root parameters:

```bash
nvim /etc/cmdline.d/root.conf
# To get UUID in nvim): =system("blkid -s UUID -o value /dev/nvme0n1p2")
```

```conf
rd.luks.name=<UUID>=root root=/dev/mapper/root rootflags=subvol=/@ rw
```

(Optional) Create a file for kernel security parameters:

```bash
nvim /etc/cmdline.d/security.conf
```

```conf
# enable apparmor
lsm=landlock,lockdown,yama,integrity,apparmor,bpf audit=1 audit_backlog_limit=8192
```

(Only if you used apparmor in security.conf) Enable apparmor:

```bash
systemctl enable apparmor.service
```

Edit the preset file to setup mkinitcpio to generate unified kernel image:

```bash
nvim /etc/mkinitcpio.d/linux.preset
```

```conf
# mkinitcpio preset file for the 'linux' package

#ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux"

PRESETS=('default' 'fallback')

#default_config="/etc/mkinitcpio.conf"
#default_image="/boot/initramfs-linux.img"
default_uki="/efi/EFI/Linux/arch-linux.efi"
default_options="--splash=/usr/share/systemd/bootctl/splash-arch.bmp"

#fallback_config="/etc/mkinitcpio.conf"
#fallback_image="/boot/initramfs-linux-fallback.img"
fallback_uki="/efi/EFI/Linux/arch-linux-fallback.efi"
fallback_options="-S autodetect"
```

Generate the unified kernel image:

```bash
mkdir -p /efi/EFI/Linux
mkinitcpio -P
```

#### Setup Boot Loader

##### Option 1: Directly from UEFI

TODO: Check configuration without boot loader.

##### Option 2: systemd-boot

Install systemd-boot:

```bash
bootctl install
systemctl enable systemd-boot-update.service
```

Edit the loader configuration:

```bash
nvim /efi/loader/loader.conf
```

```conf
default @saved
timeout 0
console-mode auto
editor no
```

### Reboot into the new system

First setup password for root, so that we can login after reboot:

```bash
passwd
```

Exit from chroot, unmount the partitions and reboot, go into UEFI firmware after reboot to follow along next steps:

```bash
exit
umount -R /mnt
reboot
```

### Enable secure boot

Once inside UEFI firmware, delete existing secure boot keys to set secure boot into setup mode. Then create new keys and enroll them:

```bash
sbctl status # verify sbctl not installed, secure boot disabled and in setup mode
sbctl create-keys
sbctl enroll-keys -m # -m for enrolling microsoft keys as well, system may get bricked if they are needed but not enrolled
sbctl status # verify sbctl installed
sbctl verify # to find files to sign
sbctl sign -s /efi/EFI/Linux/arch-linux.efi
sbctl sign -s /efi/EFI/Linux/arch-linux-fallback.efi
sbctl sign -s /efi/EFI/BOOT/BOOTX64.EFI
sbctl sign -s /efi/EFI/systemd/systemd-bootx64.efi
sbctl sign -s -o /usr/lib/systemd/boot/efi/systemd-bootx64.efi.signed /usr/lib/systemd/boot/efi/systemd-bootx64.efi
```

Reboot & enable secure boot in UEFI firmware:

```bash
sbctl status # verify sbctl installed, secure boot enabled
```

### Enrolling LUKS key into TPM2

Now that secure boot is enabled, we can enroll the LUKS key into TPM2 for automatic disk unlock at boot:

```bash
systemd-cryptenroll /dev/nvme0n1p2 --recovery-key # securely store the obtained recovery key somewhere
systemd-cryptenroll /dev/nvme0n1p2 --wipe-slot=empty --tpm2-device=auto --tpm2-pcrs=7
```

Reboot and verify LUKS is auto unlocked.

## Graphics Driver Installation

This part is very much dependent on the PC, follow official [wiki](https://wiki.archlinux.org/title/Xorg#Driver_installation) page.

Here the installation is done for Integrated Intel GPU and Discrete NVIDIA GPU.

Todo: need to verify this part and add steps for Nvidia Optimus.

```bash
pacman -S mesa mesa-utils lib32-mesa vulkan-intel lib32-vulkan-intel nvidia-open nvidia-utils lib32-nvidia-utils
mkdir -p /etc/pacman.d/hooks
nvim /etc/pacman.d/hooks/nvidia.hook
```

```conf
[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=nvidia-open
Target=linux

[Action]
Description=Updating NVIDIA module in initcpio
Depends=mkinitcpio
When=PostTransaction
NeedsTargets
Exec=/bin/sh -c 'while read -r trg; do case $trg in linux*) exit 0; esac; done; /usr/bin/mkinitcpio -P'
```

## Desktop Environment

TODO: need to decide which desktop environment to use.
