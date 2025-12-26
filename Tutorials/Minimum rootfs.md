# Minimum rootfs

Ref: https://unix.stackexchange.com/questions/136278/what-are-the-minimum-root-filesystem-applications-that-are-required-to-fully-boo



That entirely depends on what services you want to have on your device.

## Programs

You can make Linux boot directly into a **[shell](https://en.wikipedia.org/wiki/Unix_shell)**. It isn't very useful in production — who'd just want to have a shell sitting there — but it's useful as an intervention mechanism when you have an interactive bootloader: pass `init=/bin/sh` to the kernel command line. All Linux systems (and all unix systems) have a Bourne/POSIX-style shell in `/bin/sh`.

You'll need a set of **shell utilities**. [BusyBox](http://www.busybox.net/) is a very common choice; it contains a shell and common utilities for file and text manipulation (`cp`, `grep`, …), networking setup (`ping`, `ifconfig`, …), process manipulation (`ps`, `nice`, …), and various other system tools (`fdisk`, `mount`, `syslogd`, …). BusyBox is extremely configurable: you can select which tools you want and even individual features at compile time, to get the right size/functionality compromise for your application. Apart from `sh`, the bare minimum that you can't really do anything without is `mount`, `umount` and `halt`, but it would be atypical to not have also `cat`, `cp`, `mv`, `rm`, `mkdir`, `rmdir`, `ps`, `sync` and a few more. BusyBox installs as a single binary called `busybox`, with a symbolic link for each utility.

The first process on a normal unix system is called **[`init`](http://en.wikipedia.org/wiki/Init)**. Its job is to start other services. BusyBox contains an init system. In addition to the `init` binary (usually located in `/sbin`), you'll need its configuration files (usually called `/etc/inittab` — some modern init replacement do away with that file but you won't find them on a small embedded system) that indicate what services to start and when. For BusyBox, `/etc/inittab` is optional; if it's missing, you get a root shell on the console and the script `/etc/init.d/rcS` (default location) is executed at boot time.

That's all you need, beyond of course the programs that make your device do something useful. For example, on my home router running an [OpenWrt](http://en.wikipedia.org/wiki/OpenWrt) variant, the only programs are BusyBox, `nvram` (to read and change settings in NVRAM), and networking utilities.

Unless all your executables are statically linked, you will need the dynamic loader (`ld.so`, which may be called by different names depending on the choice of libc and on the processor architectures) and all the **dynamic libraries** (`/lib/lib*.so`, perhaps some of these in `/usr/lib`) required by these executables.

## Directory structure

The [Filesystem Hierarchy Standard](http://refspecs.linuxfoundation.org/FHS_2.3/fhs-2.3.html) describes the common directory structure of Linux systems. It is geared towards desktop and server installations: a lot of it can be omitted on an embedded system. Here is a typical minimum.

- `/bin`: executable programs (some may be in `/usr/bin` instead).
- `/dev`: device nodes (see below)
- `/etc`: configuration files
- `/lib`: shared libraries, including the dynamic loader (unless all executables are statically linked)
- `/proc`: mount point for the [proc filesystem](http://en.wikipedia.org/wiki/Procfs)
- `/sbin`: executable programs. The distinction with `/bin` is that `/sbin` is for programs that are only useful to the system administrator, but this distinction isn't meaningful on embedded devices. You can make `/sbin` a symbolic link to `/bin`.
- `/mnt`: handy to have on read-only root filesystems as a scratch mount point during maintenance
- `/sys`: mount point for the [sysfs filesystem](http://en.wikipedia.org/wiki/Sysfs)
- `/tmp`: location for temporary files (often a `tmpfs` mount)
- `/usr`: contains subdirectories `bin`, `lib` and `sbin`. `/usr` exists for extra files that are not on the root filesystem. If you don't have that, you can make `/usr` a symbolic link to the root directory.

### Device files

Here are some typical entries in a minimal `/dev`:

- `console`
- `full` (writing to it always reports “no space left on device”)
- `log` (a socket that programs use to send log entries), if you have a [`syslogd`](http://en.wikipedia.org/wiki/Syslog) daemon (such as BusyBox's) reading from it
- [`null`](http://en.wikipedia.org/wiki//dev/null) (acts like a file that's always empty)
- `ptmx` and a [`pts` directory](https://unix.stackexchange.com/questions/93531/what-is-stored-in-dev-pts-files-and-can-we-open-those), if you want to use [pseudo-terminals](http://en.wikipedia.org/wiki/Pseudo_terminal) (i.e. any terminal other than the console) — e.g. if the device is networked and you want to telnet or ssh in
- `random` (returns random bytes, risks blocking)
- `tty` (always designates the program's terminal)
- `urandom` (returns random bytes, never blocks but may be non-random on a freshly-booted device)
- `zero` (contains an infinite sequence of null bytes)

Beyond that you'll need entries for your hardware (except network interfaces, these don't get entries in `/dev`): serial ports, storage, etc.

For embedded devices, you would normally create the device entries directly on the root filesystem. High-end systems have a script called `MAKEDEV` to create `/dev` entries, but on an embedded system the script is often not bundled into the image. If some hardware can be hotplugged (e.g. if the device has a USB host port), then `/dev` should be managed by **[udev](http://en.wikipedia.org/wiki/Udev)** (you may still have a minimal set on the root filesystem).

## Boot-time actions

Beyond the root filesystem, you need to mount a few more for normal operation:

- [procfs](http://en.wikipedia.org/wiki/Procfs) on `/proc` (pretty much indispensible)
- [sysfs](http://en.wikipedia.org/wiki/Sysfs) on `/sys` (pretty much indispensible)
- `tmpfs` filesystem on `/tmp` (to allow programs to create temporary files that will be in RAM, rather than on the root filesystem which may be in flash or read-only)
- tmpfs, devfs or devtmpfs on `/dev` if dynamic (see udev in “Device files” above)
- [devpts](https://unix.stackexchange.com/questions/93531/what-is-stored-in-dev-pts-files-and-can-we-open-those) on `/dev/pts` if you want to use [pseudo-terminals (see the remark about `pts` above)

You can make an [`/etc/fstab`](http://en.wikipedia.org/wiki/Fstab) file and call `mount -a`, or run `mount` manually.

Start a [syslog](http://en.wikipedia.org/wiki/Syslog) daemon (as well as `klogd` for kernel logs, if the `syslogd` program doesn't take care of it), if you have any place to write logs to.

After this, the device is ready to start application-specific services.

## How to make a root filesystem

This is a long and diverse story, so all I'll do here is give a few pointers.

The root filesystem may be kept in RAM (loaded from a (usually compressed) image in ROM or flash), or on a disk-based filesystem (stored in ROM or flash), or loaded from the network (often over [TFTP](http://en.wikipedia.org/wiki/TFTP)) if applicable. If the root filesystem is in RAM, make it the [initramfs](http://en.wikipedia.org/wiki/Initramfs) — a RAM filesystem whose content is created at boot time.

Many frameworks exist for assembling root images for embedded systems. There are a few pointers in the [BusyBox FAQ](http://www.busybox.net/FAQ.html#build_system). [Buildroot](http://buildroot.uclibc.org/) is a popular one, allowing you to build a whole root image with a setup similar to the Linux kernel and BusyBox. [OpenEmbedded](http://www.openembedded.org/) is another such framework.

Wikipedia has an (incomplete) list of popular [embedded Linux distributions](http://en.wikipedia.org/wiki/Category:Embedded_Linux_distributions). An example of embedded Linux you may have near you is the [OpenWrt](http://en.wikipedia.org/wiki/OpenWrt) family of operating systems for network appliances (popular on tinkerers' home routers). If you want to learn by experience, you can try [Linux from Scratch](http://www.linuxfromscratch.org/), but it's geared towards desktop systems for hobbyists rather than towards embedded devices.

## A note on Linux vs Linux kernel

The only behavior that's baked into the Linux kernel is that the first program that's launched at boot time. (I won't get into [initrd](http://en.wikipedia.org/wiki/Initrd) and [initramfs](http://en.wikipedia.org/wiki/Initramfs) subtleties here.) This program, traditionally called [init](http://en.wikipedia.org/wiki/Init), has process ID 1 and has certain privileges (immunity to [KILL signals](http://en.wikipedia.org/wiki/SIGKILL#SIGKILL)) and responsibilities (reaping [orphans](http://en.wikipedia.org/wiki/Orphan_process)). You can run a system with a Linux kernel and start whatever you want as the first process, but then what you have is an operating system based on the Linux kernel, and not what is normally called “Linux” — [Linux](http://en.wikipedia.org/wiki/Linux), in the common sense of the term, is a [Unix](http://en.wikipedia.org/wiki/Unix)-like operating system whose kernel is the [Linux kernel](http://en.wikipedia.org/wiki/Linux_kernel). For example, Android is an operating system which is not Unix-like but based on the Linux kernel.



----



**Minimal init hello world program step-by-step**

As shown in this answer, all you need is a single statically linked ELF file without even the standard library, therefore a filesystem with a single file.

[![enter image description here](https://i.sstatic.net/rKGUn.png)](https://i.sstatic.net/rKGUn.png)

Compile a hello world without any dependencies that ends in an infinite loop. `init.S`:

```
.global _start
_start:
    mov $1, %rax
    mov $1, %rdi
    mov $message, %rsi
    mov $message_len, %rdx
    syscall
    jmp .
    message: .ascii "FOOBAR FOOBAR FOOBAR FOOBAR FOOBAR FOOBAR FOOBAR\n"
    .equ message_len, . - message
```

We cannot use `sys_exit`, or else the kernel panics.

Then:

```
mkdir d
as --64 -o init.o init.S
ld -o init d/init.o
cd d
find . | cpio -o -H newc | gzip > ../rootfs.cpio.gz
ROOTFS_PATH="$(pwd)/../rootfs.cpio.gz"
```

This creates a filesystem with our hello world at `/init`, which is the first userland program that the kernel will run. We could also have added more files to `d/` and they would be accessible from the `/init` program when the kernel runs.

Then `cd` into the Linux kernel tree, build is as usual, and run it in QEMU:

```
git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux
git checkout v4.9
make mrproper
make defconfig
make -j"$(nproc)"
qemu-system-x86_64 -kernel arch/x86/boot/bzImage -initrd "$ROOTFS_PATH"
```

And you should see a line:

```
FOOBAR FOOBAR FOOBAR FOOBAR FOOBAR FOOBAR FOOBAR
```

on the emulator screen! Note that it is not the last line, so you have to look a bit further up.

You can also use C programs if you link them statically:

```
#include <stdio.h>
#include <unistd.h>

int main() {
    printf("FOOBAR FOOBAR FOOBAR FOOBAR FOOBAR FOOBAR FOOBAR\n");
    sleep(0xFFFFFFFF);
    return 0;
}
```

with:

```
gcc -static init.c -o init
```

You can run on real hardware with a USB on `/dev/sdX` and:

```
make isoimage FDINITRD="$ROOTFS_PATH"
sudo dd if=arch/x86/boot/image.iso of=/dev/sdX
```

Great source on this subject: http://landley.net/writing/rootfs-howto.html It also explains how to use `gen_initramfs_list.sh`, which is a script from the Linux kernel source tree to help automate the process.

**Minimal setup that gives you a shell**

Buildroot is my favorite option, see discussion at: [What is the smallest possible Linux implementation?](https://unix.stackexchange.com/questions/2692/what-is-the-smallest-possible-linux-implementation/203902#203902)

At this point you are basically obliged to deal with the standard library, what insane mind who would code a `sh` shell without a standard library? So you are better off just using some automation script to set all of that up.

Tested on Ubuntu 16.10, QEMU 2.6.1.