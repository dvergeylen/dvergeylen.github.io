---
layout: post
series_title: "Self hosted Nextcloud 20+ on a Single Board Computer: a complete guide"
toc_title: "PART 2"
title:  "[2/5] Self hosted Nextcloud 20+"
date:   2021-03-23 12:30:00 +0200
tag: nextcloud
permalink: /self-hosted-nextcloud-on-sbc-complete-guide-part2
---

## See also
* [PART 1](/self-hosted-nextcloud-on-sbc-complete-guide-part1)
* PART 2 ← You are here 🙂
* [PART 3](/self-hosted-nextcloud-on-sbc-complete-guide-part3)
* PART 4
* [PART 5](/self-hosted-nextcloud-on-sbc-complete-guide-part5)

## Read only root
A read only root partition is very useful when running a web server from a single board computer, as it will dramatically decrease the number of writes on the SDCard (which is the usual point-of-failure). Inexperienced users will usually encounter no problem installing and exposing a web server to the Internet but will most probably see serious damages on their SD Card a few months later. This is due to incoming Internet traffic (legit one but also bots, ...) that will lead to spurious writes on the SDCard. Setting the root file system as read only non only prevents such writes but also avoids the necessity to track which processes are trying to write whatever information.

"Read only" is a bit of an overstatement though, as we'll actually set an `overlayfs`, two partitions used for the same purpose. The former to read new chunks of data (the original `/`) and the latter as a `tmpfs` filesystem where the OS will write them back.

(→ It actually goes the other way round: the OS tries to find data from the `tmpfs` partition and if nothing is found, falls back to read the information from the overlayed (read-only) partition.

When rebooted, the OS will loose the data written to `tmpfs` partition and start from a known state (untouched `/`).

#### Ensuring kernel has `overlayfs` as a module
We'll use the `overlayroot` script from [`overlayroot`](https://packages.ubuntu.com/groovy/overlayroot) package from Ubuntu. At this time of writing, `overlayroot` [is not available in the Debian packages](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=860915)) although its source package [cloud-initramfs-tools](https://tracker.debian.org/pkg/cloud-initramfs-tools) is still maintained.

`overlayroot` needs `overlayfs` to be compiled as a module, *not* in hard in the kernel. To ensure this is the case, check the target's current kernel config ([see here](https://superuser.com/a/287372)).

```bash
grep OVERLAY_FS /boot/config-5.10.0-4-amd64
# [...]
# CONFIG_OVERLAY_FS=m
# [...]
```

* If you see the same output as above, you can skip the next section.
* If you see `# CONFIG_OVERLAY_FS is not set` or `CONFIG_OVERLAY_FS=y`, then it's not available at all, or not compiled as a module, respectively. You need to configure `overlayfs` properly and compile a kernel yourself.

#### Compiling a vanilla Kernel from kernel.org 😎
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
                         # 5.10   LTS until Dec, 2022
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
# overlayfs is in 'File systems' → 'Overlay Filesystem Support' and must be set to 'M'

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

##### Double check
Boot your device with the (properly umounted from your HOST machine) SD Card and check if overlay is indeed available:
```bash
grep overlay /proc/filesystems
# Outputs:
nodev	overlay
```

🥳 Congratulations, your now have a freshly compiled kernel on your SD Card with `overlayfs` support! 🎉

#### Installing and configuring `overylayroot`
`overlayroot` is a script that automatically mounts and overlayFS `/`, mounting a `tmpfs` as updir.

As said above, `overlayroot` is not directly available in Debian packages, so we download it from Ubuntu package repositories.

```bash
# ON THE TARGET MACHINE
sudo apt install cryptsetup cryptsetup-bin
wget http://fr.archive.ubuntu.com/ubuntu/pool/main/c/cloud-initramfs-tools/overlayroot_0.47ubuntu1_all.deb
sudo dpkg -i overlayroot_0.47ubuntu1_all.deb

# Don't edit /etc/overlay.conf as it's part of the package
# Edit/Create /etc/overlayroot.local.conf instead and add the following:
overlayroot="tmpfs:swap=1,recurse=0"

# tmpfs → updir is a tmpfs
# swap:1 → allows mounting swaps listed in /etc/fstab (otherwise they are left unmounted)
# recurse=0 → do not mount the child partitions in overlay mount.
# For instance, if /home is a distinct partition it won't be mounted in overlay mode (same for /boot, /media, etc)
```
This will mount original `/` in `/media/root-ro` and a (tmpfs) updir in `/media/root-rw`, an overlay in `/media/root-rw/overlay` and a workdir un `/media/root-rw/overlay-workdir`.

##### Rebooting
After reboot, inspecting `mount`'s output gives:

```bash
mount
# [...]
# /dev/mmcblk0p2 on /media/root-ro type ext4 (rw,relatime,data=ordered)
# tmpfs-root on /media/root-rw type tmpfs (rw,relatime)
# overlayroot on / type overlay (rw,relatime,lowerdir=/media/root-ro,upperdir=/media/root-rw/overlay,workdir=/media/root-rw/overlay-workdir/_)
# [...]
# /dev/mmcblk0p1 on /boot type vfat (rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=iso8859-1,shortname=mixed,errors=remount-ro)
```

Yeah 😎!

## Strengthening the read only
Although overlayFS doesn't make more than necessary use of SDCard, `overlayroot` mounts it in `/media/root-ro` with default options. This includes `rw` and `relatime`, which doesn't prevent writes on the partition.

To enforce the read-only status of the parititon, we can write a script to remount `/media/root-ro` in read only mode if appears in `mount` output.

```bash
# ⚠️ be sure to deactivate overlayroot before executing this command,
# otherwise your modifications will be lost after reboot!
# (comment /etc/overlayroot.local.conf and reboot)

sudo vim /media/root-ro/root/remount_root_ro.sh

# Add the following content:
#!/usr/bin/env sh
if [ $(mount | grep -c "/media/root-ro") -gt 0 ] ; then
  mount -o remount,defaults,noatime,ro -t ext4 /dev/mmcblk0p2 /media/root-ro
fi
# End of file content

# Restrict the permissions to 'root' user:
sudo chmod 744 /media/root-ro/root/remount_root_ro.sh # rwx r-- r--

# Install cron:
sudo apt install cron
sudo -u root crontab -e
# add the following line:
# @reboot /media/root-ro/root/remount_root_ro.sh
```

After reboot, we now have:
```bash
mount
# [...]
# /dev/mmcblk0p2 on /media/root-ro type ext4 (ro,noatime,data=ordered)
# [...]
```
Which is indeed read-only! 😎


## Next

See [PART 3](/self-hosted-nextcloud-on-sbc-complete-guide-part3)