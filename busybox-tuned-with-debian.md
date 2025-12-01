# busybox - tuned with `debian:bookworm`



## The Three Parts

To build a minimal linux distro, we need three parts:
1. The Kernel
2. Userspace (busybox)
3. Bootloader (syslinux)

When the system boots, it loads the kernel, which loads busybox.
```
Bootloader -> Kernel -> Userspace
```



## Create a Container

It's bad to [install locallly](https://www.youtube.com/watch?v=J0NuOlA2xDc), so we'll do everything in an Ubuntu container.
```sh
docker run --privileged --name busybox-test -it debian:bookworm
```
You'll notice that `--privileged` flag. It is required for mounting some stuff later on.



### Setup the Container (inside the docker container)

Start by updating the system:
```sh
apt update
```

Then we need to install some packages.
```sh
apt install bzip2 unzip git vim make gcc libncurses-dev flex bison bc cpio libelf-dev libssl-dev
```



## Build Linux from Source

First, we need to get a kernel.

We'll start by cloning linux. We'll use the GitHub mirror, because why not?
```sh
mkdir linux
#git clone --depth 1 https://github.com/torvalds/linux.git ## rejected me due to I'm in North Africa?
# wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.15.176.tar.xz && tar -xf linux-5.15.176.tar.xz -C linux --strip-components=1

wget https://github.com/torvalds/linux/archive/refs/tags/v6.8.zip && unzip v6.8.zip linux --strip-components=1
cd linux
```
`--depth 1` just means that it won't clone the entire git history. (That would be significantly longer)

Next, we need to make the kernel config. There's a lot of options, and I'm sure you can find some other tutorial for making a kernel config. For this, we'll just use the default options.
```sh
make menuconfig
```
Then arrow over to `< Exit >`, and save the configuration.

So, without further ado, let's build the kernel.
```sh
make -j$(nproc)
```
This will take a LONG time...

At the end, you should see a line that says:
```txt
Kernel at arch/x86/boot/bzImage is ready
```

This is our kernel binary. Since we'll need it later, let's copy it to another place.
```sh
mkdir /boot-files
cp arch/x86/boot/bzImage /boot-files/
```



## Build the Userspace (Busybox)

Busybox is a bundle of many basic linux utilities, for example `ls`, `grep`, and `vi` are all included.

First, let's clone the sources (this process will feel very similar to the Kernel)
```sh
cd ../
mkdir busybox
# curl -L https://git.busybox.net/busybox/snapshot/busybox-1_37_0.tar.bz2 |tar xj -C busybox --strip-components=1
git clone --depth 1 https://git.busybox.net/busybox
cd busybox
```

Now, we can edit the busybox config. 

```shell
make menuconfig
```

In this one, we will need to change one option. Go to `Settings --->` and toggle `[ ] Build static binary (no shared libraries)`. This will make our build simpler; not depending on any shared libraries.

Then exit, and save changes, just like in the kernel.

Now we'll build it, just like the kernel:
```sh
make -j$(nproc)
```
It shouldn't take nearly as long, but you never know...

Since busybox will go into our initramfs, let's make a folder for it in `/boot-files`:
```sh
mkdir /boot-files/initramfs
```

Now, we'll install busybox to that directory:
```
make CONFIG_PREFIX=/boot-files/initramfs install
```
This will basically just create a bunch of symlinks in that folder

Also, we can remove `linuxrc`, because we don't need it.
```sh
rm /boot-files/initramfs/linuxrc
```



### Create an init script

The kernel needs something to execute. It looks in `/init` for this file. On a normal linux system, this is a systemd binary, but here, it's just a shell script.
```sh
cd /boot-files/initramfs
echo << EOF > ./init
#!/bin/sh

/bin/sh
EOF
chmod +x ./init
```
This short little script will launch a shell as soon as we boot into the system.



## Create the Initramfs

An initramfs is a cpio archive. This archive needs to contain all of the files in `initramfs/`. First, let's create a list of all of those files:
```sh
find .
```

Then pass it to `cpio` to create the archive:
```sh
find . | cpio -o -H newc > ../init.cpio
```
`-o` creates a new archive, and `-H newc` specifies the type of the archive.



## Create the "filesystem"

We don't want to create a whole partition for this device, and, luckily, we don't have to. We'll create an empty file and format it with fat.
```sh
dd if=/dev/zero of=./boot bs=1M count=50
```
Hopefully you can spare 50M...

To format it, we'll need `dosfstools`:
```sh
apt install -y dosfstools
```

Then to format it:
```sh
mkfs -t fat boot
```



### Install the bootloader

We'll need the `syslinux` package:
```sh
apt install -y syslinux
```

Then we can install syslinux with:
```sh
syslinux ./boot
```



### Copy files to the image

We need to mount that image (Here's why our container needed root privileges). Create the mountpoint, then mount it there.
```sh
mkdir m
mount boot m
```

Then move the kernel binary and the initramfs to the image:
```sh
cp {bzImage,init.cpio} m/
```

Now we can umount the filesystem:
```sh
umount m
```

Now exit to the host:

```shell
exit
```



## Test it

We can test the image using qemu. Assuming you don't want to install qemu on your container, we'll copy the files to the host system.
```sh
mkdir ~/boot-files
# docker cp {container_name}:/boot-files ~/boot-files
docker cp busybox-test:/boot-files ~/
```

Now we can run `./boot` with qemu:
```sh
cd ~/boot-files/initramfs/
ls -la
# apt Install qemu-system-x86
qemu-system-x86_64 -drive format=raw,file=./boot -display gtk
```

We'll get a prompt that says `boot: `. Type,
```sh
/bzImage -initrd=/init.cpio
```

Just wait for it to boot up, then you'll be greeted with a shell prompt!
```
~ # 
```



## Bonus: Installing binaries

You can install any software that you want to the system using the following procedure:
1. Build the software, with `CONFIG_PREFIX=/boot-files/initramfs`
2. Remount `./boot` to `m`
3. Delete the old `init.cpio` and `m/init.cpio`
4. Create a new one, and copy it to `m/`
5. Umount `./m`