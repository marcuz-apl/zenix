# Install Alpine to a Partition

To install Alpine Linux on a partition, boot the live ISO, run `setup-alpine` for basic setup (network, hostname), then manually partition your target disk (e.g., with `cfdisk`), format the partitions (e.g., `mkfs.ext4`), mount the root partition to `/mnt`, run `setup-disk -m sys /mnt` to install the system and bootloader, and finally reboot to finish configuration. Key steps involve creating an EFI partition (for UEFI systems), a root partition, and optionally a swap, ensuring internet access for package downloads. 

This video provides a complete walkthrough of the Alpine Linux installation process:



### 1. Boot and Initial Setup

- Boot from the Alpine ISO and log in as `root` (no password).

- Set up your network (e.g., `setup-interfaces`, `setup-dns`).

- Run `setup-alpine` and follow prompts for hostname, root password, time zone, and APK mirror. 



### 2. Partitioning and Formatting

- Use `cfdisk`, `fdisk`, or `gdisk` to create partitions (e.g., `/dev/sda1` for EFI, `/dev/sda2` for swap, `/dev/sda3` for root).
- **For UEFI:** Create a ~300MB FAT32 EFI System Partition (ESP).
- **For Swap:** Create a Linux swap partition.
- **For Root:** Create a Linux filesystem partition (ext4, Btrfs).
- Format the partitions (e.g., `mkfs.vfat -F32 /dev/sda1`, `mkswap /dev/sda2`, `mkfs.ext4 /dev/sda3`).
- **Important:** Use `blkid` to identify your partition names (e.g., `/dev/sda3`) and UUIDs. 



### 3. Installation to Disk

- Mount your root partition: `mount /dev/sda3 /mnt` (replace `/dev/sda3`).
- If using EFI, mount the ESP: `mkdir /mnt/boot/efi && mount /dev/sda1 /mnt/boot/efi`.
- Run the disk installation script: `setup-disk -m sys /mnt` (ensure `BOOTLOADER=grub` is set in environment if needed).
- This installs base packages, generates `/etc/fstab`, and installs the bootloader. 



### 4. Finalize and Reboot

- Exit the chroot environment when prompted.
- Reboot the system: `reboot`.
- Remove the installation media and boot into your new Alpine Linux system. 