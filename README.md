## Overview

The goal of this is to document my steps for installing Gentoo on my machine. In general, I want to install a [Distribution Kernel](https://wiki.gentoo.org/wiki/Distribution_Kernel), KDE plasma, OpenRC, and a [Binary Host Setup](https://wiki.gentoo.org/wiki/Gentoo_Binary_Host_Quickstart).

I'm open to critique and general pointers for improvements as I document this. Of course I will use the [Gentoo Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64) as my primary reference.

## Installing on Virtual Machine

I want to start installing this on a Virtual Machine. I'm doing this on a CachyOS host with a QEMU+KVM virtual machine:

### <ins>Getting ISO</ins>

First thing is to get the AMD64 minimalist ISO from [Gentoo's website](https://www.gentoo.org/downloads/amd64/).

After acquiring the ISO, I created a new VM using [Virtual Machine Manager](https://virt-manager.org/):
- CPUs: 4
- RAM: 8 GiB
- Storage: 400 GiB
- Video: VirtIO
- Display: SPICE
    - ![VMM Display](./Images/VMM-Display-SPICE.png)


After installing the VM, I get this screen:
- ![ISO Boot Screen](./Images/Gentoo-ISO-Boot.png)

Select "Boot LiveCD (kernel: gentoo)
- ![First Terminal](./Images/First-Terminal.png)

### <ins>Network Configuration</ins>

- Test the virtual network interface: `ping -c3 wiki.gentoo.org`
    - In case for some reason, this test doesn't work, make sure that there's a working virtual network interface:
        - ![Virtual Network Interface](./Images/Virtual-Network-Interface.png)
    - Or setup */etc/resolv.conf* like this:
        - ![DNS Resolver Configuration](./Images/Resolv.conf.png)

### <ins>Disk Setup</ins>

Assuming networking works, move onto disk setup. Because I'm on a virtual machine, I'm using a virtualized disk (`/dev/vda`).

![Virtual Disk](./Images/Virtual-Disk.png)

I'm using `fdisk` (Format Disk) to format and partition the virtual disk. I'll use this graph as a guide (except I want to use [btrfs](https://docs.kernel.org/filesystems/btrfs.html) for my root filesystem instead of xfs).

![Fdisk Guide](./Images/Filesystem-Format-Guide.png)

This is the series of commands I used to format my disk:
1. `fdisk /dev/vda`
    - Starts setup for /dev/vda
2. `n`
    - Creates a new partition
3. `p`
    - Partition Type: Primary
4. `1`
    - Creates partition #1
5.  Press Enter
    - Starts Partition 1 at wherever is the next available sector
6. `+1G`
    - This means that Partition 1 starting at sector 2048 will be of size 1 GiB
7. `t`
    - We're selecting the type of our partition
8. `ef`
    - 0xEF corresponds to "EFI (FAT12/16/32)"
9. `n`
    - Creates a new partition
10. `p`
    - Partition Type: Primary
11. `2`
    - Creates partition #2
12. Press Enter
    - Starts Partition 1 at wherever is the next available sector
13. `+16G`
    - I have 8GiB of RAM, so I'll use 16 GiB for the swap. Likely a bit much for this system, but I'm lazy and I don't care lol
14. `t`
    - We're selecting the type of our partition
15. `2`
    - Selects partition #2
16. `82`
    - 0x82 corresponds to "Linux swap"
17. `n`
    - Creates a new partition
18. `p`
    - Partition Type: Primary
19. `3`
    - Creates partition #3
20. Press Enter
    - Starts Partition 1 at wherever is the next available sector
21. Press Enter
    - Use the rest of the disk for the root partition
22. `w`
    - Writes to the partition table and creates the partitions on the disk

Now to actually format the partitions:

- EFI Partition (/dev/vda1)
    - `mkfs.vfat -F32 /dev/vda1`
        - This means to "make filesystem that is FAT32 on /dev/vda1 (the first partition)"
- Swap Partition (/dev/vda2)
    - `mkswap /dev/vda2`
    - `swapon /dev/vda2`
- Root Partition (/dev/vda3)
    - `mkfs.btrfs /dev/vda3`

Now to mount the root and boot partitions:

1. `mkdir -p /mnt/gentoo`
2. `mount /dev/vda3 /mnt/gentoo`
3. `mkdir -p /mnt/gentoo/boot`
4. `mount /dev/vda1 /mnt/gentoo/boot`

### <ins>Stage File Installation</ins>

The partitions created and mounted. It's time to get the stage file for Gentoo, which is an archive containing all necessary files to run a minimal Gentoo build. To acquire it:

1. `cd /mnt/gentoo`
2. `links https://www.gentoo.org/downloads/mirrors/`
    - Scroll down to the regions screen:
        - ![Regions Screen](./Images/Regions-Links-Screen.png)
    - Pick a mirror URL (I just picked the first one)
        - ![Mirror URL](./Images/Links-Mirror-URL.png)
    - Go to *releases* --> *amd64* --> *autobuilds* --> *current-stage3-amd64-desktop-openrc*
    - Download the latest .tar.xz file
        - ![Stage3 Tarball](./Images/Latest-Stage3-Tarball.png)
3. `tar -xpvf ./stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner -C /mnt/gentoo`

### <ins>Base System Installation</ins>

Now that I have the base files for our Gentoo build, time to configure some options for Portage, Gentoo's package management system:

- `cp --dereference /etc/resolv.conf /mnt/gentoo/etc/`
- `vim /mnt/gentoo/etc/portage/make.conf`
```
# Global Compiler Flags
COMMON_FLAGS="-march=native -O2 -pipe"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON FLAGS}"
FFLAGS="${COMMON_FLAGS}"
RUSTFLAGS="${RUSTFLAGS} -C target-cpu=native"
MAKEOPTS="-j4 -l5"
NINJAOPTS="-j4"

# Locale Settings
LC_MESSAGES=C.UTF-8
```

Of course, I will add to the make.conf later. For now, it's time to add the other filesystems for /proc, /sys, /dev/, and /run:
- /proc: A pseudo-filesystem for the Linux kernel
- /sys: A pseudo-filesystem like /proc and was designed to be the successor to /proc
- /dev: A regular filesystem which contains all devices and is managed by a device manager (usually `udev`)
- /run: A tempoarary filesystem used for files generated at runtime, such as PID files and locks

1. `mount --types proc /proc /mnt/gentoo/proc`
2. `mount --rbind /sys /mnt/gentoo/sys`
3. `mount --make-rslave /mnt/gentoo/sys`
4. `mount --rbind /dev /mnt/gentoo/dev`
5. `mount --make-rslave /mnt/gentoo/dev`
6. `mount --bind /run /mnt/gentoo/run`
7. `mount --make-slave /mnt/gentoo/run`

Now that the virtual filesystems are installed, it's time to chroot (Change Root) to the install location (/mnt/gentoo):

1. `chroot /mnt/gentoo /bin/bash`
2. `source /etc/profile`
3. `export PS1="(chroot) ${PS1}`
    - The screen should now look like this:
        - ![Chroot](./Images/Chroot.png)

### <ins>How To Resume Install</ins>

This is a good stopping point here and I'll include this little section here since I know I will be on and off from this install. This section will show how to get back in the install root after turning off the VM because I'm trying to have a life lol

1. `mount /dev/vda3 /mnt/gentoo`
2. `mount /dev/vda1 /mnt/gentoo/boot`
3. `swapon /dev/vda2`
4. `mount --types proc /proc /mnt/gentoo/proc`
5. `mount --rbind /sys /mnt/gentoo/sys`
6. `mount --make-rslave /mnt/gentoo/sys`
7. `mount --rbind /dev /mnt/gentoo/dev`
8. `mount --make-rslave /mnt/gentoo/dev`
9. `mount --bind /run /mnt/gentoo/run`
10. `mount --make-slave /mnt/gentoo/run`
11. `chroot /mnt/gentoo /bin/bash`
12. `source /etc/profile`
13. `export PS1="(chroot) ${PS1}`

### <ins>Portage Configuration</ins>

### <ins>Kernel Configuration</ins>

### <ins>System Installation: Finishing Touches</ins>

### <ins>Network Configuration</ins>

## Installing on Real Hardware

[TBD]
