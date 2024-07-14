---
title: Breathe life into old computers with a minimal Linux installation
tags: 
- Linux
- Minimalism
- optimization
desc: Don't throw out old computers, put Linux on them.
layout: post
---

Minimalism is quite a rarity these days. Instead of focusing on optimizations that can be done on the current state of someone's hardware, people choose to just buy better compute units.
You certainly don't need an RTX 4090 for programming. In fact, you don't need anything more than some integrated graphics^[1], which basically every laptop these days gets shipped with. 
<!-- more -->

It would be inherently wrong to conclude that *every laptop* is fine for programming. The Operating System on it matters a lot. The criticisms of the obvious shortcomings of some mainstream OSes is out the scope of this article, so I'll make a whole new page for it.

If you're an impatient dev, start right from the next paragraph. But, if you must know the reasons (which you should), read the heading "Why Linux?" first.

# Disclaimer
Arch, which will be used here, is NOT a beginner friendly distribution. It comes with barely any source modifications, and minimalism is its [core philosophy](https://wiki.archlinux.org/title/Arch_Linux#Principles).

But, if that deters you from trying it out, you can try Arch on a [Virtual Machine](https://itsfoss.com/install-arch-linux-virtualbox/), which is the safest option that doesn't compromise any data.
# Prerequisites
All you need is a stable internet connection, and a USB stick. The storage on the stick doesn't matter, the installation file is less than a gig.

On Windows:
- Download [Balena Etcher](https://etcher.balena.io/). It is the simplest way to flash a Linux ISO onto your USB.

On Linux:
- Well, why are you reading if you already have Linux installed? Anyhow, you don't need anything. Linux ships with the [dd](https://www.man7.org/linux/man-pages/man1/dd.1.html) command, that is sufficient.

# Flashing Linux
Grab the latest release for Arch Linux^[2] from [here](https://archlinux.org/download/). Just scroll down and download the ISO from the mirror closest to you.

- On Balena, you just need to select the ISO file wherever you downloaded it, and Flash it to the USB stick. Be careful though, you could accidentally flash it to some other drive, which will completely wipe it.
In most cases, this flash is permanent, meaning you can't recover any data.

- On Linux:
1) Identify the USB stick:
```bash
$ lsblk -ap 
NAME             MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
/dev/sdb           8:16   1  57.3G  0 disk
├─/dev/sdb1        8:17   1   950M  0 part
└─/dev/sdb2        8:18   1   164M  0 part
```

It's usually named something like `/dev/sdb`^[3] where the [partitions](https://en.wikipedia.org/wiki/Disk_partitioning) on it are named `/dev/sdbX` where X` is a number. 

2) Flash the ISO to the stick:
```bash
$ sudo dd if=/path/to/iso/archlinux-xxx.iso  of=/dev/sdb1 bs=4M status=progress
```
`of` is the output file, `if` is the source ISO, `bs` declares the byte size to read at a time (default is `512` bytes, but I set it to `4M` for faster reads).

- et voilà, it's done. no additional software needs to be installed.

### Reboot
When you restart your computer, enter the BIOS, and boot from the USB stick.

# Command-line installation
You'll see a terminal interface if you've followed along carefully. It is time to setup Arch.
PS: Most of this is condensed from the [ArchWiki](https://wiki.archlinux.org/title/Installation_guide). If you get errors, the fix for them is always there.

1) The font is usually small, let's change it.
```
# setfont ter-132n
```

NOTE: if you find this font too large, switch to some other, smaller font:
```
# ls /usr/share/kbd/consolefonts/ | grep -i ter-1
```
or
```
# ls /usr/share/consolefonts/ | grep -i ter-1
```
and switch to any font in the list. Omit the `pcf.gz` name:
```
# setfont ter-118n
```

## Internet
2) Connect to the internet:
```
# iwctl
# station wlan0 connect <WIFI_NAME>
# exit
```

2.5) Test your connection:
```
# ping archlinux.org
```

## Partitioning
It's important to configure your disk and create sections on it, where different parts of Linux are used.
You can do everything in one partition, but it's usually not a good idea. 

4) Use [cfdisk](https://en.wikipedia.org/wiki/Cfdisk) to partition as follows:
I'll assume you're using an old hard disk, in which case it will be named something like `/dev/sdaX`.

I use this partition table:
```
/dev/sda1 -> EFI -> 1GB
/dev/sda2 -> [SWAP]* -> 8GB
/dev/sda3 -> / (root file system)
```
- if you have a single partition on the disk, click on `Resize`,  and you will get some Free Space. Do this 3 times to get 3 different partitions.
- Once you get the 3 partitions, `[ Write ]` the changes to disk.

4.5)  For SWAP Memory, set the size to twice the RAM installed in your computer. You can check it:
```bash
$ free -ht
```
And see the section under `Total` and the row `Mem`.

PS: It is not necessary to have SWAP, but it is [strongly recommended](https://www.linux.com/news/all-about-linux-swap-space/).

5) Now that you have 3 partitions on your disk, it's time to format them.
**NOTE:** This wipes all the data on your disk.

Following the partition scheme above, format them as follows:
```
# mkfs.fat -F 32 /dev/sda1
# mkswap /dev/sda2
# mkfs.ext4 /dev/sda3
```

6) [Mount](https://en.wikipedia.org/wiki/Mount_%28computing%29) the partitions:

The root partition:
```
# mount /dev/sda3  /mnt
```

The boot partition:
```
# mount --mkdir /dev/sda1 /mnt/boot
```

Enable the Swap partition:
```
# swapon /dev/sda2
```

PS: [/mnt](https://tldp.org/LDP/Linux-Filesystem-Hierarchy/html/mnt.html)is a generic mount point. Essentially, it's just a folder. It's not necessary that you mount things relative to `/mnt`, but it is suggested to use it so that you don't clutter your filesystem with mountpoints. 

Also note that `--mkdir` will *create* the `/mnt/boot` directory. Alternatively, you can create the boot directory beforehand:
```
# mkdir /mnt/boot
# mount /dev/sda1 /mnt/boot
```

## Installing the system

7) Install the core packages: the [base](https://archlinux.org/packages/?name=base), the [Linux Kernel](https://www.kernel.org/) and [firmware](https://en.wikipedia.org/wiki/Firmware):
```
# pacstrap -K /mnt base base-devel linux-zen linux-firmware nano vim
```
[linux-zen](https://github.com/zen-kernel/zen-kernel) is an official Kernel version, which is supposed to be the best for daily driving linux, plus it boasts of [optimized desktop usage, low latency](https://github.com/zen-kernel/zen-kernel/wiki/Detailed-Feature-List) and much more.

[Vim](https://en.wikipedia.org/wiki/Vim_(text_editor)i and [Nano](https://en.wikipedia.org/wiki/GNU_nano) are console text editors^[5]. These allow you to edit files without even installing a [Graphical User Interface](https://en.wikipedia.org/wiki/Graphical_user_interface).
Read more about the choice for these editors in the footnootes.

8) Generate an [fstab](https://wiki.archlinux.org/title/Fstab):
```
# genfstab -U /mnt >> /mnt/etc/fstab
```

9)  [Chroot](https://en.wikipedia.org/wiki/Chroot)^[4] (`"change root"`) into your brand new installation:
```
# arch-chroot /mnt
```

Essentially, now you can access files and utilities present within your system, aka `/mnt`.
`Chroot`-ing is a very important process. I've written more about it in the footnotes.

10) Set the timezone and generate a locale:
```
# ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
# hwclock --systohc
```
For example, `Asia/Calcutta` for which the path is: `/usr/share/zoneinfo/Asia/Calcutta`.

11) Create a [hostname](https://en.wikipedia.org/wiki/Hostname):
```
# nano /etc/hostname
```
And add your host name. For example, I have a laptop called `Mozart`:
```
/etc/hostname
-----------------

Mozart
```
This host name appears on things such as bluetooth connections.

11.5) Set a locale:
```
/etc/locale.conf
-----------------
LANG=en-US.UTF-8
```
```
# locale-gen
```

12) Complete your network connection, so that Wi-Fi starts as soon as you boot the pc next time.
```
[root@archiso /]# pacman -S networkmanager grub
```
[Pacman](https://wiki.archlinux.org/title/Pacman) helps you install literally *everything* on Arch.

[NetworkManager](https://wiki.archlinux.org/title/NetworkManager) is a program that has the most support for different connection types, and mostly works out of the box.
[Grub](https://en.wikipedia.org/wiki/GNU_GRUB) is the most commonly used [boot loader](https://en.wikipedia.org/wiki/Bootloader).

Enable the network manager:
```
# systemctl enable NetworkManager
```

Set a [root](https://en.wikipedia.org/wiki/Superuser) password:
```
# passwd
```
"Root" is the Windows equivalent of an "Administrator Account".

## Bootloader configuration
Install GRUB for your entire drive:
```
# grub-install /dev/sda
```

or, if you're using an NVMe drive,
```
# grub-install /dev/nvme0nX
```
where `X` is the number for the NVMe you're using.

Generate a configuration for GRUB:
```
# grub-mkconfig -o /boot/grub/grub.cfg
```

## Setup the minimal desktop
Create a user account, say "shiven" with admin priveleges:
```
# useradd -m shiven
```
To be able to access `sudo` commands for superusers, it must be added to the `sudoers` list
```
# EDITOR=nano visudo
```
and change the line:
```
USER_NAME   ALL=(ALL:ALL) ALL
```
to:
```
shiven   ALL=(ALL:ALL) ALL
```


Install a browser (Firefox), a [Window Manager](https://en.wikipedia.org/wiki/Window_manager) ([i3](https://i3wm.org/)), a [Terminal emulator](https://en.wikipedia.org/wiki/Terminal_emulator) ([Alacritty](https://en.wikipedia.org/wiki/Alacritty)), Media Player ([VLC](https://www.videolan.org/vlc/)), a [Shell](https://en.wikipedia.org/wiki/Unix_shell) ([zsh](https://en.wikipedia.org/wiki/Z_shell)), and a File Manager ([Thunar](https://en.wikipedia.org/wiki/Thunar)).
```
# pacman -S i3wm alacritty firefox vlc thunar zsh 
```

# Finishing up
Exit the `chroot` environment:
```
# exit
```

Un-mount all the directories: (Always recommended)
```
# umount -R /mnt
```

and that's it. This is probably the most bloat-free Linux installation. 

I've used this exact configuration for my Raspberry Pi 5, and it works like a charm. No lag, no overheating (except when using the browser).
