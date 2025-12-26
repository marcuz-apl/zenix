# Make minimum rootfs bootable

ref: https://www.reddit.com/r/AlpineLinux/comments/13ochmy/how_to_make_mini_rootfs_bootable/

To complete the setup, btw thanks to [u/ncopa](https://www.reddit.com/user/ncopa/)'s comment, here's the following commands I've used:

- Create root and boot partitions for UEFI/GPT (substitute `X` with the actual device):

```
cfdisk /dev/sdX
mkfs.vfat -F32 /dev/sdX1
mkfs.ext4 -O "^has_journal,^64bit" /dev/sdX2
```

- Mount the partitions:

```
mount /dev/sdX2 /mnt
mkdir /mnt/boot
mount /dev/sdX1 /mnt/boot
```

- Extract the mini rootfs (Alpine edge):

```
wget -O - https://dl-cdn.alpinelinux.org/alpine/edge/releases/x86_64/alpine-minirootfs-20230329-x86_64.tar.gz | tar -C /mnt -xzpf -
```

> Configure mountpoints in `/mnt/etc/fstab` (try to use the output from command `mount | grep '/mnt'`).

> Configure `nameserver` in `/mnt/etc/resolv.conf` for networking (try to copy the contents of host's configuration).

- Mount host filesystem and enter chroot environment:

```
for fs in dev dev/pts proc run sys tmp; do mount -o bind /$fs /mnt/$fs; done
chroot /mnt /bin/sh -l
```

- Install the base, kernel and bootloader packages:

```
apk add --update alpine-base linux-edge grub grub-efi efibootmgr
```

- Configure GRUB bootloader:

```
grub-install --target=x86_64-efi --efi-directory=/boot
```

> Add `--no-bootsector --removable` if the device is a portable drive.

- Configure OpenRC init system (https://gitlab.alpinelinux.org/alpine/mkinitfs/-/blob/dcb90f4bb5c7b7749c405be27d101774b559643e/initramfs-init.in#L676):

```
rc-update add devfs sysinit
rc-update add dmesg sysinit
rc-update add mdev sysinit
rc-update add hwdrivers sysinit

rc-update add modules boot
rc-update add sysctl boot
rc-update add hostname boot
rc-update add bootmisc boot
rc-update add syslog boot

rc-update add mount-ro shutdown
rc-update add killprocs shutdown
rc-update add savecache shutdown

rc-update add firstboot default
```

> Refer to [Wi-Fi](https://wiki.alpinelinux.org/wiki/Wi-Fi) wiki for networking.

> Refer to [Setting up new user](https://wiki.alpinelinux.org/wiki/Setting_up_a_new_user) wiki for user account.

- Reboot and test it out.

> If `root` partition is failed to mount on start, try what [u/strawbeguy](https://www.reddit.com/user/strawbeguy/) mentioned:
>
> - Add `GRUB_CMDLINE_LINUX="modules=ext4 rootfstype=ext4"` in `/etc/default/grub`,
> - Then run `grub-mkconfig -o /boot/grub/grub.cfg` to update GRUB configuration.