---
layout: post
series_title: "Self hosted Nextcloud 20+ on a Single Board Computer: a complete guide"
toc_title: "PART 1"
title: "[1/5] Self hosted Nextcloud 20+"
date: 2020-10-12 12:30:00 +0200
tag: nextcloud
permalink: /self-hosted-nextcloud-on-sbc-complete-guide-part1
last_update: 2021-09-08 12:30:00 +0200
---

<p class="info">
  📚 Installation on a single board computer: a complete guide
</p>

## Introduction
In this long post, I'll describe how to configure a single board computer such as the **Raspberry Pi** or the **Asus Tinker Board** from A to Z to support a self-hosted Nextcloud 20+ instance. We'll go through the following steps:

* [PART 1 - Installing a recent Operating System (Debian 11 - bullseye)](#installing-a-recent-image):
  * [Preliminaries](#installing-a-recent-image)
  * [Copying the Bootloader](#copying-the-bootloader)
  * [Installing a recent Kernel from kernel.org (5.4 LTS supported until December 2025)](#installing-a-recent-kernel-from-kernelorg)
  * [Installing Debian 11 Bullseye](#installing-debian-11-bullseye) (`debootstrap`)
* [PART 2 - Readonly root](/self-hosted-nextcloud-on-sbc-complete-guide-part2):
  * [Partitioning a read-only root](/self-hosted-nextcloud-on-sbc-complete-guide-part2#read-only-root) (via `overlayroot`)
  * [Strengthening the read-only](/self-hosted-nextcloud-on-sbc-complete-guide-part2#strengthening-the-read-only)
* [PART 3](/self-hosted-nextcloud-on-sbc-complete-guide-part3):
  * [Installing + configuring MariaDB](/self-hosted-nextcloud-on-sbc-complete-guide-part3#installing-mariadb)
  * [**Installing Nextcloud 20+**](/self-hosted-nextcloud-on-sbc-complete-guide-part3#installing-nextcloud)
  * Installing and configuring [Nginx](/self-hosted-nextcloud-on-sbc-complete-guide-part3#installing-and-configuring-nginx) or [Apache](/self-hosted-nextcloud-on-sbc-complete-guide-part3#installing-and-configuring-apache) in front of it
  * [Tuning and tweaking Nextcloud 20+ freshly installed](/self-hosted-nextcloud-on-sbc-complete-guide-part3#configuring-nextcloud)
* [PART 4](/self-hosted-nextcloud-on-sbc-complete-guide-part4):
  * [Installing Let's Encrypt SSL wildcard certificates](/self-hosted-nextcloud-on-sbc-complete-guide-part4#lets-encrypt-tls-certificate)
  * [Dynamic host via `ddclient`](/self-hosted-nextcloud-on-sbc-complete-guide-part4#dynamic-host-via-ddclient)
* [PART 5](/self-hosted-nextcloud-on-sbc-complete-guide-part5):
  * [Remounting as a read/write FS](/self-hosted-nextcloud-on-sbc-complete-guide-part5#deactivate-read-only-root)
  * [Updating Nextcloud](/self-hosted-nextcloud-on-sbc-complete-guide-part5)
  * [**Backuping your SD Card**]() (do that regularly while going through this guide)

My personal board is an Asus Tinker Board, but I've played with Raspberry Pi 4s as well before, and the procedure is roughly the same (>90%). I'll describe the differences when they arise (mainly at the image installation and cleanup steps).

## Installing a recent image

#### Raspberry pi
Download [Raspberry Pi OS (32-bit) Lite](https://www.raspberrypi.org/downloads/raspberry-pi-os/) image.

Unzip the file; you'll end up with an .img file like `YYY-MM-DD-raspios-buster-armhf-lite.img`. The image is a dump of a 2Gb partition with ≃ 435Mb of content.

#### Tinker board
Download the latest image from [here](https://www.asus.com/uk/Single-Board-Computer/Tinker-Board/HelpDesk_Download/). The latest one for TinkerBoard 1 is the **2.1.16**. Starting at **2.2.x** you must at least have a Tinker Board S (with UMS support).

Unzip the file; you'll end up with an .img file like `Tinker_Board-Debian-Stretch-V2.1.16-20200813.img`. The image is a dump of a 2Gb partition with an old Kernel (4.04) and an old Debian stretch 9. We'll update that shortly, and only keep the bootloader.

#### Putting the image on the SD Card

To transfer the `.img` content from your hard drive to the SDCard, copy its content via the `dd` command (Linux). Be aware you have to know which device name your SD card receives from the OS, **your mileage may (will) vary**. The best way to know is probably to `ls /dev` folder, then insert the SD Card, and `ls` it again. The newly added device is your SD Card (e.g. `/dev/sdb`). If your SD Card has multiple partitions, they will appear with a number as suffix like `/dev/sdb1`, `/dev/sdb2`, etc. We are only interested in the device name (e.g. `/dev/sdb`), **not** the partition name(s).

```bash
# Transfer the .img content to SD Card
sudo dd bs=4M if=Tinker_Board-Debian-Stretch-V2.1.16-20200813.img of=/dev/sdb status=progress && sync
```
⌛ This will take some time, be patient. 

Once done, starting `gparted` can be handy to extend the copied partition. SD Cards have > than 16Gb by now, and the copied image is a 2Gb partition.

If you have a raspberry pi, you are good to go and can directly jump to [Part 2](/self-hosted-nextcloud-on-sbc-complete-guide-part2).

## Copying the Bootloader

#### U-boot vanilla?
Theoretically, the TinkerBoard is officially supported by u-boot and we should compile it like this:

```bash
cd /tmp
git clone https://source.denx.de/u-boot/u-boot.git
cd u-boot

make CROSS_COMPILE=arm-linux-gnueabihf- tinker-rk3288_defconfig
# If you want to see U-boot outputs, set SILENT_CONSOLE option to Yes via menuconfig

make CROSS_COMPILE=arm-linux-gnueabihf- -j15
```

By convention, UART2 is used by Rockchip for serial output: pin 32 for TX, 33 for RX and 30 as GND. Do not connect Vcc, prefer an external power supply.

Transfer it on the SD card via the following command:
```bash
sudo dd if=u-boot-rockchip.bin of=/dev/mmcblk0 seek=64
sync
```

The problem is: via this technique, U-boot has never been able to find the kernel on the first partition. I can see U-boot's output but I get a weird error. Please contact me (via the comments below) if you were able to configure the sources to make it boot nicely, I would be more than happy to have a recent u-boot on my board!

#### U-boot from Asus fork?
As we can't compile a fresh u-boot, let's compile the sources from Asus's fork! Located [here](https://github.com/TinkerBoard/debian_u-boot). I tried to compare the modifications done on the fork with the original repo but the two are too far away now and many options disappeared, changed their name, or have been added. I could find a reliable way to compare and set the same fork's configuration on u-boot sources.

In addition to that, [you need an old Ubuntu distribution](https://github.com/TinkerBoard/debian_u-boot/issues/8#issuecomment-812232236) to make it compile, otherwise your build will fail.

#### U-boot from the given image
Our last chance is to use the already compiled u-boot binary from the given image. We can't recreate it but we can use it! If you copied the given image on an SD card, you have nothing to do. U-boot binary is already at sector 64.


## Installing a recent Kernel from kernel.org
Let the fun begin!

Download the kernel sources and compile them on the host machine, don't do this on the target.
```bash
# ON THE HOST MACHINE

# Installing required compilers
sudo apt install gcc-arm-linux-gnueabihf device-tree-compiler gcc-aarch64-linux-gnu bc

# Downloading the vanilla kernel:
git clone https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
cd linux
git branch --list --remotes
git branch --show-current
git checkout linux-5.4.y # 5.4 is LTS until Dec, 2025
                         # 5.10   LTS doesn't boot with Asus defconfig
                         # See: https://www.kernel.org/category/releases.html

# Download kernel configuration for the target board
# Asus gives a file for Tinkerboard / TinkerBoard S / TinkerBoard 2
curl -L https://raw.githubusercontent.com/TinkerBoard/debian_kernel/develop/arch/arm/configs/miniarm-rk3288_defconfig -o arch/arm/configs/miniarm-rk3288_defconfig

# Configure the kernel sources with the downloaded configuration file
make ARCH=arm miniarm-rk3288_defconfig -j16

# (a couple of warnings might appear but they should be harmless)
# We now need to add 'overlayfs' support as a module.
# The below command will start a terminal GUI, where you can browse the kernel config options
make ARCH=arm menuconfig
# overlayfs is at 'File systems' → 'Overlay Filesystem Support' and must be set to 'M'
# hardware crypto module support is at ' Cryptographic API' → 'Hardware crypto devices' → 'Rockchip's Cryptographic Engine driver' and must be set to 'M'

# Compile kernel and its modules
make zImage ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j16
make modules ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j16

# Tinker's dts is in mainline kernel, under the name rk3288-tinker.dts
# (see: arm/boot/dts/rk3288-tinker.dts)
# so compile it as well to a dtb file (binary)
make ARCH=arm rk3288-tinker.dtb CROSS_COMPILE=arm-linux-gnueabihf- -j16

# Install modules to the SDCard's '/' partition (adapt 'sdX' below)
sudo make ARCH=arm INSTALL_MOD_PATH=/media/sdX/ modules_install

# Copy freshly compiled kernel to the SDcard's '/boot' partition (adapt 'sdY' below)
cp arch/arm/boot/zImage /media/sdY/zImage
cp arch/arm/boot/dts/rk3288-tinker.dtb /media/sdY/rk3288-miniarm.dtb
```

🥳 Congratulations, your now have a freshly compiled kernel on your SD Card with `overlayfs` and `rk_crypto` support! 🎉


## Installing Debian 11 Bullseye
For this part, you'll need to format (yes, format) the second partition made by copying the .img file. Prefer EXT4 filesystem.

Once done, mount the partition in a temporary folder:
```bash
mkdir /tmp/os
sudo mount /dev/sdb2 /tmp/os
```

And use `deboostrap` to install a fresh Debian stable. (`apt install debootstrap` might be necessary).

```bash
# First Stage (downloading packages)
sudo /usr/sbin/debootstrap --arch armhf --foreign stable /tmp/os https://deb.debian.org/debian/

# Second stage (unpacking packages in a chrooted env):
# as the target arch is different, needed to use qemu to chroot
sudo apt install qemu-user-static
sudo cp /usr/bin/qemu-arm-static /tmp/os/usr/bin/
sudo chroot /tmp/os /usr/bin/qemu-arm-static /bin/bash -i

# When chrooted, run the second stage
#  /debootstrap/debootstrap --second-stage
```

#### Minimal configuration
Still in the chrooted environment, 

```bash
# Update root password
passwd

# Specify hostname
nano /etc/hostname

# Modify FSTAB
# Add the following lines (mind the tabs):
# /dev/mmcblk0p2	/	ext4	defaults	0	1
# /dev/mmcblk0p1	/boot	vfat	defaults	0	2
nano /etc/fstab 

# Enable serial console
systemctl enable serial-getty@ttyS0.service

# Configure locales
apt install locales

apt update
apt install git openssh-server vim firmware-linux firmware-linux-nonfree firmware-linux-free firmware-realtek firmware-iwlwifi cron acl build-essential debian-keyring initramfs-tools autoconf automake libtool sudo ufw curl lsb-release cryptsetup cryptsetup-bin unzip curl

useradd -m sheeva
echo "sheeva:monpassword" | chpasswd
adduser sheeva sudo
dpkg-reconfigure locales
vim /etc/ssh/sshd_config
```

##### Upgrading Source list:
update `/etc/apt/sources.list` with the following content:

```conf
deb http://deb.debian.org/debian/ stable main contrib non-free
deb-src http://deb.debian.org/debian/ stable main contrib non-free

deb http://deb.debian.org/debian/ stable-updates main contrib non-free
deb-src http://deb.debian.org/debian/ stable-updates main contrib non-free

# Added from https://wiki.debian.org/Backports#Using_the_command_line
deb http://deb.debian.org/debian bullseye-backports main contrib non-free
```

##### Setting up a static IP address
As this SBC will become a Nextcloud server, it's better to assign it a fixed IP. Add the following in `/etc/network/interfaces.d/eth0` (file won't exist):
```
auto eth0
iface eth0 inet static
address 192.168.1.XXX
netmask 255.255.255.0
network 192.168.1.0
broadcast 192.168.1.255
gateway 192.168.1.1
```
`/etc/network/interfaces` imports `/etc/network/interfaces.d/*`, names doesn't matter.
Replace `XXX` with the actual IP address you want to assign your board.

##### Securing SSH
Only allow connection via `$MYUSER`.

```bash
sudo vim /etc/ssh/sshd_config

# Add/Modify the following directives:
AllowUsers MYUSER
PermitRootLogin no
```
Other ssh tweaks can be found [here](https://www.digitalocean.com/community/tutorials/ssh-essentials-working-with-ssh-servers-clients-and-keys).

🥳 Congratulations, your now have a freshly installed Debian stable on your SD Card! 🎉

## Next

See [PART 2](/self-hosted-nextcloud-on-sbc-complete-guide-part2)