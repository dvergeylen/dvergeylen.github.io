---
layout: post
series_title: "Self hosted Nextcloud 20+ on a Single Board Computer: a complete guide"
toc_title: "PART 2"
title:  "[2/5] Self hosted Nextcloud 20+"
date:   2021-03-23 12:30:00 +0200
tag: nextcloud
permalink: /self-hosted-nextcloud-on-sbc-complete-guide-part2
last_update: 2021-09-08 12:30:00 +0200
---

## See also
* [PART 1](/self-hosted-nextcloud-on-sbc-complete-guide-part1)
* PART 2 ← You are here 🙂
* [PART 3](/self-hosted-nextcloud-on-sbc-complete-guide-part3)
* [PART 4](/self-hosted-nextcloud-on-sbc-complete-guide-part4)
* [PART 5](/self-hosted-nextcloud-on-sbc-complete-guide-part5)

## Read only root
A read only root partition is very useful when running a web server from a single board computer, as it will dramatically decrease the number of writes on the SDCard (which is the usual point-of-failure). Inexperienced users will usually encounter no problem installing and exposing a web server to the Internet but will most probably see serious damages on their SD Card a few months later. This is due to incoming Internet traffic (legit one but also bots, ...) that will lead to spurious writes on the SDCard. Setting the root file system as read only non only prevents such writes but also avoids the necessity to track which processes are trying to write whatever information.

"Read only" is a bit of an overstatement though, as we'll actually set an `overlayfs`, two partitions used for the same purpose. The former to read new chunks of data (the original `/`) and the latter as a `tmpfs` filesystem where the OS will write them back.

(→ It actually goes the other way round: the OS tries to find data from the `tmpfs` partition and if nothing is found, falls back to read the information from the overlayed (read-only) partition.

When rebooted, the OS will loose the data written to `tmpfs` partition and start from a known state (untouched `/`).

#### Ensuring kernel has `overlayfs` as a module
We'll use the `overlayroot` script from [`overlayroot`](https://packages.ubuntu.com/groovy/overlayroot) package from Ubuntu. At this time of writing, `overlayroot` [is not available in the Debian packages](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=860915)) although its source package [cloud-initramfs-tools](https://tracker.debian.org/pkg/cloud-initramfs-tools) is still maintained.

`overlayroot` needs `overlayfs` to be compiled as a module, *not* in hard in the kernel. To ensure this is the case, check `/proc/filesystems`.

```bash
# Load the overlayfs module:
sudo modprobe overlay

grep overlay /proc/filesystems
# Outputs:
# nodev	overlay
```

* If you see the same output as above, you can skip the next section.
* If you don't see the correct output or loading the module fails, you'll have to recompile the kernel with the appropriate options, see Part 1.


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

##### Troubleshooting
If you don't see the output above, you might be missing an initrd script. To create one, run

```bash
export PATH="/usr/sbin:$PATH"
sudo update-initramfs -c -k "$(uname -r)"
```

This will create an initrd file in `/boot`. To tell u-boot to use it on startup, edit `/boot/extlinux/extlinux.conf` file and add a 'initrd' config line:

```conf
  label kernel-5.4.144+
  kernel /zImage
  fdt /rk3288-tinker.dtb
  initrd /initrd.img-5.4.144+
  append  earlyprintk splash console=ttyS3 console=tty1 rw init=/sbin/init
```

Replace 'initrd' suffix by your kernel version.

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


## Hardware Crypto acceleration support (CONFIG_CRYPTO_DEV_ROCKCHIP)
To enable support of the hardware crypto acceleration, one will set `CONFIG_CRYPTO_DEV_ROCKCHIP` to 'm'. The option can be found at at ' Cryptographic API' → 'Hardware crypto devices' → 'Rockchip's Cryptographic Engine driver' in `make ARCH=arm menuconfig` (forgetting `ARCH=arm` won't display the option).

Ensure module loads at every boot time by editing `/etc/modules-load.d/modules.conf`:
```bash
# /etc/modules: kernel modules to load at boot time.
#
# This file contains the names of kernel modules that should be loaded
# at boot time, one per line. Lines beginning with "#" are ignored.
rk_crypto
```

##### Double check
Boot your device and check via `lsmod | grep crypto` (should output `rk_crypto`). 🎉

## Next

See [PART 3](/self-hosted-nextcloud-on-sbc-complete-guide-part3)