---
title: Linux system in an hour or less
date: 2020-10-14T00:00:00+02:00
---
{{<quote "why the lucky stiff">}}When you don’t create things, you become defined by your tastes rather than ability. Your tastes only narrow and exclude people. so create.{{</quote>}}

Consider the following question from section [II: Audience](http://www.linuxfromscratch.org/lfs/view/stable/prologue/audience.html) of [Linux From Scratch (LFS) project](http://www.linuxfromscratch.org):

> Why go through all the hassle of manually building a Linux system from scratch when you can just download and install an existing one?

I don't have a compelling reason myself, and chances are, neither do you. But it's fun, so let's build a linux system from scratch, for sheer entertainment.

We will go through building a very minimal linux system that just works to get an idea of how things work and interact together, we will build more advanced systems in later articles, but for now, let's stick with the very basics.

In order to build a (minimal) Linux system we need to have:
* a Linux kernel,
* system libraries,
* system utils, and
* a shell

In the following sections we will briefly discuss what each componenet is responsible for and why do we need it.

### Linux kernel

The kernel is the first part of the system that is loaded in memory, starts working and loads the rest of the system. It's a program responsible for bootstrapping and managing the resources of the system.

The kernel abstracts away most hardware details so application developers wouldn't have to worry if they are reading files from a FAT32 partition on a spinning harddisk or reading a file from network over an InfiniBand connection.

### System Libraries

The kernel exposes an ABI to applications that can be used to interact with the system components like devices, file systems, processes, networking, etc. However, the ABI exposed by the kernel is minimal and only offers the smallest possible set of routines required to operate the system.

For more advanced system routines/functionality, we have user-space libraries. These libraries offer extra functionality not offered by the kernel directly, like compression, cryptography, rendering, etc.

The most important library is the C Standard Library `libc`. This library acts as a glue/wrapper over kernel routines and offers some extra functionality. Almost all applications use `libc`. We will be using [musl-libc](http://musl.libc.org/), which is a small yet correct C Standard.


### System Utils

While system libraries offer enough functionality to use the system, we still need to write code to use them. System utils is a set of common applications that most people would need, such as `cat`, `ls`, `cp`, `mv`, etc.

These utils are pre-written and can be used directly by users to execute common operations. In our case we will be using [BusyBox](https://busybox.net/) which is a single binary that has many utils and can be used to build a small system.

### Shell
Finally, we need a shell. The shell is yet another program that is used by users directly to execute commands, combine programs to create pipelines, and offers some scripting interface.

Luckily, BusyBox already provides a shell and we will be using it.

## Setting up the environment
Create a directory to store everything related to this article, we will have a lot of files.

We will be using [qemu](https://www.qemu.org/) to test the system we build, make sure to install it.

We use [docker](https://www.docker.com/) to create a build environment that is consistent across distros.
This is a minimal Dockerfile to create a docker image that includes all the packages required to build the system:
```Dockerfile
FROM alpine
RUN apk add build-base gcc ncurses-dev xz
```

Build the image
```bash
$ docker build -t lfs-build .
```

Create a directory called `src/` to store all the source packages we will need along with a few directories under `/src/`
```bash
$ mkdir -p src/{boot,sysroot}
```

Run the contaienr and mount `src/` inside it:
```bash
$ docker run --rm -it -v $(pwd)/src:/src lfs-build
```

All the following commands should be executed inside the container (unless otherwise specified).

## Building the kernel

Head to [kernel.org](https://www.kernel.org/) and fetch a kernel tarball of any (recent) kernel version. I'm using [4.14.200](https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.14.200.tar.xz).

Download the tarball to `/src/` and extract it
```bash
$ wget https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.14.200.tar.xz -P /src/
$ cd /src; tar xJf linux-4.14.200.tar.xz
$ cd linux-4.14.200
```

The kernel is configured through a configuration file called `.config`, the file contains a set of params starting with `CONFIG_` that we can alter to enable/disable some kernel parts/modules and set some pre-defined values.

> Most distros provide the config used to build the kernel you're currently running, it's usually `/boot/config-$(uname -r)`, you can check it to know what params were used in your kernel.

The default config for `x86_64` is rather huge and takes a lot of time to build. Since we only interest ourselves in a small system, we can disable most parts.

Instead of starting from scratch or starting from the full default config, we will start with a `tinyconfig` bundled in the kernel. This is a minimal configuration that boots the kernel, but doesn't do much, we will use it as a starting point for our configuration. To generate a tiny `.config` file execute the following command:
```bash
$ make tinyconfig
```
You can then inspect `.config` file and see which configurations values are set, it's okay if you don't know/understand most of these values, but your luckiest guess is likely correct.


While this config by itself is enough to boot the kernel, it's not enough for our purposes, we will need to edit a few parts. The kernel has a nice UI for easily editing configuration, execute the following command:
```bash
$ make menuconfig
```

You should be presented with a screen inside the terminal, use the following block as a guide to what params we need to enable, leave everything else as is.

```
[*] 64-bit kernel

Device Drivers --->
    Character devices --->
        [*] Enable TTY

General setup --->
    [*] Initial RAM filesystem and RAM disk (initramfs/initrd) support
    -*- Configure standard kernel features (expert users) --->
        [*] Enable support for printk

Executable file formats / Emulations --->
    [*] Kernel support for ELF binaries
    [*] Kernel support for scripts starting with #!
```

Here's a breakdown if the params and why we need them:
* `64-bit kernel`: the default `tinyconfig` builds a 32-bit kernel, most machines are 64-bit nowadays so we enable it.
* `Enable TTY`: We need TTY support in order to run applications that need a terminal.
* `Initial RAM filesystem and RAM disk`: Instead of having to support block devices, SCSI drivers, Filesystems, we will just package our entire system as a ramdisk image and load it in memory directly.
* `Enable support for printk`: We enable `printk` support so we would get boot log messages and other console messages printed out.
* `Kernel support for ELF binaries`: We enable `ELF` support in order to be able to run the system utils and applications.
* `Kernel support for scripts starting with #!`: We enable shebang scripts support because it's heavily used.


After saving the config, we are now ready to build the kernel, execute the following command and lean back, it shouldn't take more than a few minutes.
```bash
$ make -j$(nproc)
```

After the kernel builds successfully, we need to install the kernel headers into our system root `sysroot`, and copy the kernel image to the `boot` directory.
```bash
$ make INSTALL_HDR_PATH=/src/sysroot/usr headers_install
$ cp arch/x86/boot/bzImage /src/boot/vmlinuz
```

Now we can test our kernel, try running it from the host machine by executing the following command:
```bash
$ qemu-system-x86_64 -enable-kvm -m 128M -kernel boot/vmlinuz
```
The kernel should attempt to boot and panic with a `not syncing: No working init found.` message.

Now we need a working `init` process, which is provided by BusyBox as well. In order to build BusyBox, we need a working C Library.

## Building musl-libc
Head to [musl.libc.org](http://musl.libc.org/) and download the latest release. I'm using [musl-1.2.1](http://musl.libc.org/releases/musl-1.2.1.tar.gz).

```bash
$ wget http://musl.libc.org/releases/musl-1.2.1.tar.gz -P /src/
$ cd /src; tar xzf musl-1.2.1.tar.gz
$ cd musl-1.2.1
```

musl-libc uses a `configure` script for configuration like most normal applications. We only need to set `prefix` to `/usr`.
```bash
$ ./configure --prefix=/usr
```
After the command succeeds, we should be ready to build and install musl-libc into our `sysroot`.
```bash
$ make -j$(nproc)
$ make DESTDIR=/src/sysroot install
```

This installs musl-libc into the sysroot, try looking around inside sysroot to see how it looks like.

## Building BusyBox
We are now ready to build BusyBox! Head to [busybox.net/downloads](https://busybox.net/downloads/) and fetch the latest release. I'm using [busybox-1.32.0](https://busybox.net/downloads/busybox-1.32.0.tar.bz2).

```bash
$ wget https://busybox.net/downloads/busybox-1.32.0.tar.bz2 -P /src/
$ cd /src; tar xjf busybox-1.32.0.tar.bz2
$ cd busybox-1.32.0
```

BusyBox is configured the same way Linux is configured.
```bash
$ make menuconfig
```

We need to set a few values:
```conf
Settings --->
    [*] Build static binary (no shared libs)
    (/src/sysroot) Path to sysroot
    (/src/sysroot) Destination path for 'make install' 

Linux System Utilities --->
    [ ] eject
```

Then build and install BusyBox
```bash
$ make -j$(nproc)
$ make install
```

## Making initrd
Now that we have everything setup, we just need to create a ramdsik from our `sysroot`, ramdisks are CPIO archives (optionally compressed).
```bash
$ cd /src/sysroot; find . | cpio -oH newc > ../boot/initrd.img
```

We should now have `vmlinuz` and `initrd.img` under `/src/boot`.

## Running it
On the host machine, try running qemu with the kernel and the ramdisk we just built.
```bash
$ qemu-kvm -m 512M -kernel src/boot/vmlinuz -initrd src/boot/initrd.img
```
It should boot successfully and drop you to a shell prompt.