# busybox - tuned with `ubuntu:latest`



## The Ultimate Resources

- The Official website of Kernel Archives: www.kernel.org
- Nir Lichtman's YouTube channel:
  - `Dive into Linux kernel` - https://www.youtube.com/watch?v=D4k1Q3aHpT8&list=PL0tgH22U2S3GiMeldUs9osJCFDuqjKq_V
- Chris Titus Tech YouTube Channel:
  - Build Your Own Operating System: https://www.youtube.com/watch?v=YbXHU7W7Its
  - Creating Your Own Linux Distribution: https://www.youtube.com/watch?v=pZcbHdBs_TE
- YouTube Channel by DenshiVideo: https://www.youtube.com/watch?v=APQY0wUbBow



### The referred video

Specially for this experiment, please move to: https://www.youtube.com/watch?v=QlzoegSuIzg



## The Three Parts

To build a minimal Linux distro, we need three parts:
1. The Kernel ("bzImage")
2. Userspace (busybox based "init.cpio")
3. Bootloader (syslinux, can be grub2)

When the system boots, the bootloader loads the kernel, which loads busybox.
```
Bootloader -> Kernel -> Userspace
(syslinux or grub2) -> (vmlinuz or linux-6.1.x-generic) -> (initrd.img or initramfs)
```



## 1- Install a Ubuntu 24.04 or grab a WSL distro of Such

It's bad to install locally, so I'll try everything in a WSL distro of Ubuntu 24.04.
```sh
## Install the packages as needed
sudo apt update
sudo apt install qemu-system-x86 dosfstools bzip2 curl xz-utils unzip wget git nano make gcc flex bison bc cpio libncurses-dev libelf-dev libssl-dev dosfstools -y
```


## 2- Build Linux Kernel from Source

First, we need to get a kernel.

We'll start by cloning Linux. We'll use the GitHub mirror, because why not?
```sh
mkdir linux

git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git --depth 1 --branch v6.1.158

cd linux
```
`--depth 1` just means that it won't clone the entire git history. (That would be significantly longer)

Next, we need to make the kernel config. There's a lot of options, and I'm sure you can find some other tutorial for making a kernel config. For this, we'll just use the default options.
```sh
# make tinyconfig
make menuconfig
```
Then arrow over to `< Exit >`, and save the configuration.

Important: Don’t forget to enable `CONFIG_PVH=y` in `.config` if you want to run in QEMU.

```shell
nano .config

```

then search "PVH" to update that line to be `CONFIG_PVH=y`.

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

We can give a try:

```shell
ls -lh vmlinux
qemu-system-x86_64 -kernel ./vmlinux -nographic --append "console=tty0 console=ttyS0 panic=1 root=/dev/sda rootfstype=ext2" -m 1024 -vga none -display none -serial mon:stdio -no-reboot
```

It will panic eventually, due to unable to find an `init`.

Basically the kernel was frantically looking for an `init` executable in `/sbin`,`/etc`,`/bin`. Really just any executable (can be a shell script, or a binary executable).



## 3- Our own init

Let’s write our own init for now. The `sleep` is to prevent the kernel from panicing (it will panic if `init` exits).

```shell
## Still in `linux` folder
mkdir vfs && cd vfs
nano hello-kernel.c
```

The Content of the `hello-kernel.c` belike:

```txt
#include <stdio.h>
#include <unistd.h>

int main(){
    printf("Hello, Kernel!\n");
    sleep(1200);
    return 0;
}
```

Then compile this .c file:

```shell
gcc --static hello-kernel.c -o init
chmod +x ./init
```

Now we just need to package it up into a `cpio` format.

```shell
find .
find . | cpio -o -H newc > ../initrd.img
```

Then we try QEMU successfully:

```shell
## back to `linux folder
cd ..
## give a try
qemu-system-x86_64 -kernel ./vmlinux -nographic --append "console=tty0 console=ttyS0 panic=1 root=/dev/sda rootfstype=ext2" -hda rootfs.ext2 -m 1024 -vga none -display none -serial mon:stdio -no-reboot -initrd initrd
```

Press `Ctrl+a`, then `c` returning to `(qemu)` status, then `q <enter>` to back to the Ubuntu.

Great! But now we don’t just want our `init` to say hello, we want our `init` to be able to put us into a shell and let us navigate and do productive work! That is where [busybox](https://busybox.net/) comes in!



## 4- Build the Userspace (initramfs) using Busybox

### 4.1 Download and Compile the BusyBox 

Busybox is a bundle of many basic Linux utilities, for example `ls`, `grep`, and `vi` are all included.

First, let's clone the sources (this process is very similar to the Kernel)
```sh
## don't do stuff in the linux directory, git will be confused
cd ../
mkdir busybox

### the busybox.net is not accessible, or extremely slow these days
# git clone --depth 1 https://git.busybox.net/busybox
git clone --depth=1 https://github.com/mirror/busybox.git

# wget https://github.com/mirror/busybox/archive/refs/tags/1_36_1.tar.gz && tar -xf 1_36_1.tar.gz -C busybox --strip-components=1
# wget https://github.com/mirror/busybox/archive/refs/tags/1_36_1.zip && unzip 1_36-1.zip -d busybox

cd busybox
## Check info
git log
```

Now, we can edit the busybox config. 

```shell
make menuconfig
```

In this one, we will need to change one option. Go to `Settings --->` and toggle `[ ] Build static binary (no shared libraries)`. This will make our build simpler; not depending on any shared libraries.

Then `exit` twice, and save changes, just like in the kernel.

Now we'll build it, just like the kernel:
```sh
sudo make -j$(nproc)
```
It shouldn't take nearly as long, but you never know...



### 4.2 Install compiled busybox utilities into `initramfs` folder

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

## Check out what's left in that folder?
ls -la /boot-files/initramfs
## only 3 sub-folders left
##   bin
##   sbin
##   usr
```

### 4.3 Create an init script

The kernel needs something to execute. It looks in `/init` for this file. On a normal linux system, this is a systemd binary, but here, it's just a shell script.
```sh
cd ../linux
mkdir initrd && cd initrd
mkdir etc proc sys

cat << EOF > init
#!/bin/sh

mount -t proc proc /proc
mount -t sysfs none /sys

# https://busybox.net/FAQ.html#job_control

mknod /dev/ttyS0 c 4 64
setsid sh -c 'exec sh <dev/ttyS0 > /dev/ttyS0 2>&1'

EOF

chmod +x ./init
```
This short little script (`init`) will launch a shell as soon as we boot into the system.

## 4.4 Create the Initramfs archive

An initramfs is a `cpio archive`. This archive needs to contain all of the files in `initramfs/`. First, let's create a list of all of those files:
```sh
find .
```

Then pass it to `cpio` to create the archive saving to the parent folder (in this specific format):
```sh
find . | cpio -o -H newc > ../initrd.img
## the file name was init.cpio, i renamed as initrd.img, the same way as in Ubuntu
```
`-o` creates/outputs a new archive, and `-H newc` specifies the type of the archive.

Check what we have in the /boot-files folder now:

```shell
cd ..
ls -la
## Mainly 2 files: bzImage and initrd.img, occupying 13.620 MB diskspace
```

```txt
total 13620
drwxr-xr-x 3 root root     4096 Dec  2 09:25 .
drwxr-xr-x 1 root root     4096 Dec  2 08:38 ..
-rw-r--r-- 1 root root 11568128 Dec  2 08:31 bzImage
drwxr-xr-x 5 root root     4096 Dec  2 09:16 initramfs
-rw-r--r-- 1 root root  2360320 Dec  2 09:25 initrd.img
```



## 5- Create the "filesystem"

### 5.1 Generate an empty filesystem

We don't want to create a whole partition for this device, and, luckily, we don't have to. We'll create an empty file and format it with `fat` partition format in the capacity of 40MB.

```sh
## In the folder of
## /boot-files

dd if=/dev/zero of=./boot bs=1M count=40
```
To format it, we'll need `dosfstools`:
```sh
# apt install -y dosfstools
```

Then to format it:
```sh
mkfs -t fat ./boot
```

### 5.2 Install the bootloader (syslinux)

We'll need the `syslinux` package:
```sh
# apt install -y syslinux
```

Then we can install `syslinux` onto this `boot` file with command below:
```sh
syslinux ./boot
```

### 5.3 Copy files to the image

We need to mount that image (Here's why our container needed root privileges). Create the mountpoint, then mount it there.
```sh
mkdir m
mount boot m
```

Then move the kernel binary and the initramfs to the image:
```sh
cp {bzImage,initrd.img} m/

## Optionally check the size of the m folder
du -sh m
## It's around 14MB
```

Now we can umount the filesystem:
```sh
umount m
## Remove the m folder
rmdir m
```

Optional Check what we have in the container now (`ls -la`):

```txt
total 64820
drwxr-xr-x 3 root root     4096 Dec  2 09:41 .
drwxr-xr-x 1 root root     4096 Dec  2 08:38 ..
-rw-r--r-- 1 root root 41943040 Dec  2 09:39 boot
-rw-r--r-- 1 root root 10665760 Dec  2 08:31 bzImage
drwxr-xr-x 5 root root     4096 Dec  2 09:16 initramfs
-rw-r--r-- 1 root root  2360320 Dec  2 09:25 initrd.img
```

Now exit the Docker container (`tinyvbox`) to the host:

```shell
exit
```



## 6. Test it

We can test the image using `qemu`. Assuming you don't want to install qemu on your container, we'll copy the files to the host system.
```sh
mkdir ~/boot-files
# docker cp {container_name}:/boot-files ~/
docker cp tinyvbox:/boot-files ~/
```

Now we can run the image of `./boot` with `qemu`:
```sh
cd ~/boot-files
ls -la
## Only the boot image will be used for testing
## -rw-r--r--  1 zenusr zenusr 41943040 Dec  2 11:39 boot
## -rw-r--r--  1 zenusr zenusr 10665760 Dec  2 10:31 bzImage
## drwxr-xr-x  5 zenusr zenusr     4096 Dec  2 11:16 initramfs
## -rw-r--r--  1 zenusr zenusr  2360320 Dec  2 11:25 initrd.img

## Now test it over
qemu-system-x86_64 -drive format=raw,file=./boot -display gtk -kernel bzImage -initrd initrd.img
```

Just wait for it to boot up, then you'll be greeted with a shell prompt!

```
~ # 
```



## Bonus: Tune the kernel or install more binaries

You can install any software that you want to the system using the following procedure:
1. Spin up the stopped container: 

   ```shell
   docker start -ai tinyvbox
   ```

   you'll be at `#` prompt.

2. Tune the drivers (say `TTY`) and redo `make -j$(nproc)` and copy the `bzImage` over.

3. Install more binaries, with `CONFIG_PREFIX=/boot-files/initramfs`

4. ...

5. Re-run the `qemu` session.