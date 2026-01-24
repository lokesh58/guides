# Setting up External SSD for Homelab (Arch Linux + LUKS + Btrfs)

This guide covers setting up a **Sandisk Extreme Portable SSD (1 TB)** as secure external storage for a Dell G15 5520 running Arch Linux. We will use **LUKS encryption** for security and **Btrfs** for flexible storage management (snapshots, compression).

**Target Use Case:** Storing large media (Immich, Nextcloud) for a Docker Compose homelab.

## Prerequisites

Ensure you have the necessary tools installed:

```bash
sudo pacman -S btrfs-progs cryptsetup gptfdisk
```

## 1. Identify the Drive

Connect your SSD and identify its device identifier (e.g., `/dev/sdb`).

```bash
lsblk
```

_Look for the 1TB drive. We will refer to it as `/dev/sdX` in this guide. **Replace `sdX` with your actual identifier.** Be extremely careful, as the next steps will wipe data._

## 2. Partitioning

We will create a clean GPT partition table and a single Linux filesystem partition.

```bash
# Wipe existing filesystem signatures (optional but recommended)
sudo wipefs -a /dev/sdX

# Create a new GPT table and a partition taking 100% space
sudo gdisk /dev/sdX

# --- Within gdisk interactive menu ---
# Press 'o' to create a new empty GPT partition table (Answer 'Y' to confirm)
# Press 'n' to add a new partition
#   - Partition number: default (Enter)
#   - First sector: default (Enter)
#   - Last sector: default (Enter)
#   - Hex code or GUID: 8309 (This is the code for 'Linux LUKS')
# Press 'w' to write table to disk and exit (Answer 'Y' to confirm)
# -------------------------------------
```

_Verify partitions:_ `lsblk /dev/sdX`

## 3. Setup LUKS Encryption

Encrypt the partition (`/dev/sdX1`). You will be prompted to set a passphrase. Pick a strong one; this is your master key.

```bash
sudo cryptsetup luksFormat /dev/sdX1
```

Open the encrypted partition to format it. We'll map it to a name, e.g., `secure_homelab`.

```bash
sudo cryptsetup open /dev/sdX1 secure_homelab
```

_This creates a mapped device at `/dev/mapper/secure_homelab`._

## 4. Format with Btrfs and Create Subvolume

Format the decrypted volume with Btrfs and create a root subvolume (`@data`). This mimics standard Arch layouts and allows better snapshot management.

```bash
# Format
sudo mkfs.btrfs -L homelab_data /dev/mapper/secure_homelab

# Mount temporarily to create subvolume
sudo mount /dev/mapper/secure_homelab /mnt
sudo btrfs subvolume create /mnt/@data
sudo umount /mnt
```

## 5. Auto-Unlocking (TPM2) and Mounting

We will use the laptop's TPM 2.0 module to automatically unlock the drive at boot. This binds the drive to this specific hardware.

### A. Enroll TPM2

Use `systemd-cryptenroll` to add the TPM2 chip as a key for your LUKS volume. You will need your original passphrase.

```bash
# Verify you have a TPM2 device
ls /dev/tpmrm0

# Enroll the TPM. We bind to PCR 7 (Secure Boot state).
# This ensures it unlocks only if the boot environment is trusted.
sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7 /dev/sdX1
```

_Note: If you don't use Secure Boot, you might want to omit this flag or use `--tpm2-pcrs=0`. If you change Secure Boot keys, you'll need to use your passphrase once to re-enroll._

### B. Verify Enrollment

Check that a token slot is now occupied by systemd-tpm2.

```bash
sudo cryptsetup luksDump /dev/sdX1
```

_Look for a "Token" section mentioning systemd-tpm2._

### C. Configure Crypttab (Auto-Unlock)

Get the UUID of the **physical partition** (`/dev/sdX1`).

```bash
sudo blkid /dev/sdX1
```

Edit `/etc/crypttab`:

```bash
# <name>       <device>         <password>   <options>
secure_homelab   UUID=<YOUR_UUID> -            tpm2-device=auto
```

_The password field is set to `-` because we aren't using a file or static password._

### D. Configure Fstab (Auto-Mount)

Create a mount point.

```bash
sudo mkdir -p /mnt/data
```

Get the UUID of the **decrypted filesystem** (`/dev/mapper/secure_homelab`).

```bash
sudo blkid /dev/mapper/secure_homelab
```

Edit `/etc/fstab`:

```ini
# <file system>                     <mount point>           <type>  <options>                                                                     <dump>  <pass>
UUID=<YOUR_MAPPER_UUID>             /mnt/data   btrfs   defaults,noatime,compress=zstd:3,ssd,discard=async,space_cache=v2,subvol=@data 0       0
```

_Note: We mount the `@data` subvolume explicitly. `ssd` ensures SSD optimizations are applied even over USB._

### E. Test Configuration

Reboot your system to verify it unlocks automatically.

```bash
sudo reboot
```

## 6. Docker & Btrfs Subvolumes (Best Practice)

Instead of dumping everything in the root of the drive, use Btrfs subvolumes. This allows you to snapshot specific datasets (e.g., just Immich photos) independently.

1. **Create Subvolumes:**

   ```bash
   sudo btrfs subvolume create /mnt/data/docker_volumes
   sudo btrfs subvolume create /mnt/data/backups
   ```

2. **Set Permissions:**
   If your Docker containers run as a specific user (often PUID=1000), change ownership.

   ```bash
   # Assuming your user is 'lokesh58' (uid 1000)
   sudo chown -R 1000:1000 /mnt/data/docker_volumes
   ```

3. **Update Docker Compose:**
   Point your service volumes to this new path.

   _Example `docker-compose.yml` for Immich:_

   ```yaml
   services:
     immich-server:
       volumes:
         - /mnt/data/docker_volumes/immich/upload:/usr/src/app/upload
   ```

## 7. Maintenance

Btrfs on SSDs requires periodic maintenance (trim and scrub). Enable the Arch Linux provided timers:

```bash
# Trims SSD to maintain performance
sudo systemctl enable --now fstrim.timer

# Verifies data integrity
sudo systemctl enable --now btrfs-scrub@mnt-data.timer
```
