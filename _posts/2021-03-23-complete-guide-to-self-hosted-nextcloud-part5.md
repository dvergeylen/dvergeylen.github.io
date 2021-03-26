---
layout: post
series_title: "Self hosted Nextcloud 20+ on a Single Board Computer: a complete guide"
toc_title: "BONUS"
title:  "[5/5] Self hosted Nextcloud 20+"
date:   2021-03-23 12:30:00 +0200
tag: nextcloud
permalink: /self-hosted-nextcloud-on-sbc-complete-guide-part5
---

## See also
* [PART 1](/self-hosted-nextcloud-on-sbc-complete-guide-part1)
* [PART 2](/self-hosted-nextcloud-on-sbc-complete-guide-part2)
* [PART 3](/self-hosted-nextcloud-on-sbc-complete-guide-part3)
* [PART 4](/self-hosted-nextcloud-on-sbc-complete-guide-part4)
* PART 5 ← You are here 🙂

<p class="info">
  🏗️ This part deals with maintenance burden. Not the funniest part but always needed to ensure a healthy system. I've divided the sections as if they were 'flight rules', they aren't meant to be read sequentially.
</p>

## Remounting as a read/write FS
When updates are needed or simply updating the configuration is needed, one could need to temporarily deactivate the read only filesystem.

Although `overlayroot` package gives an `overlayroot-chroot` command, don't use it for two reasons:

1. The `/media/root-ro` directory is remounted as `ro` and changes wouldn't persist
2. Packages updates / installation might break because some operations are forbidden in chrooted environment. You might experience some `Running in chroot, ignoring request.` errors.

```bash
sudo mount -o remount,defaults,noatime,rw -t ext4 /dev/mmcblk0p2 /media/root-ro

sudo vim /media/root-ro/etc/overlayroot.local.conf
# → Add '#' before `overlayroot="tmpfs:swap=1,recurse=0"` line

sudo reboot
```

## Updating Nextcloud

1. Remount the filesystem as read/write (see above) and reboot
2. Upgrade Nextcloud from the command line:
  ```bash
  sudo -u www-data php /var/www/nextcloud/updater/updater.phar  --no-interaction
  ```
  [See Offical Doc](https://docs.nextcloud.com/server/18/admin_manual/maintenance/update.html#using-the-command-line-based-updater)
3. Remount the system in readonly mode
  ```bash
    sudo vim /etc/overlayroot.local.conf # uncomment the line
    sudo reboot
  ```

## Updating the Kernel
If you recompiled your kernel in Part 2, you'll have to recompile it regularly to keep it up to date.

1.  Compile kernel sources

    ```bash
    # On the HOST machine

    # Assuming you still have a linux git clone from part 2
    git branch # Will tell you on which branch you currently are
    git pull origin linux-5.4.y

    # Restart incremental compilation
    make zImage ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j16
    make modules ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j16

    # Compile the DTS (should not be necessary)
    make ARCH=arm rk3288-tinker.dtb CROSS_COMPILE=arm-linux-gnueabihf- -j8

    # Transfer to SDCard
    # Installing drivers on SD Card
    sudo make ARCH=arm INSTALL_MOD_PATH=/media/$USER/324cd421-b656-4326-9882-de35a3ad335a/ modules_install

    # Installing on the target's /boot partition
    cp arch/arm/boot/zImage /media/$USER/4C89-2DED/
    cp arch/arm/boot/dts/rk3288-miniarm.dtb /media/$USER/4C89-2DED/
    ```
2.  When rebooting, recreate an `initframfs`:

    ```bash
    En bootant, refaire un initrd:
    su root
    export PATH="/usr/sbin:$PATH"
    update-initramfs -c -k "$(uname -r)"
    # This will create an initrd in /boot (ex: "initrd.img-5.4.94+")
    ```
3.  You'll have to update the `/boot/extlinux/extlinux.conf` file with the two new files.


## Backing up your SD Card
1. Put your SD Card in your host pc
2. Umount it manually
3. Launch a terminal and type:
  ```bash
  sudo dd if=/dev/mmcblk0 conv=sync,noerror bs=64K | gzip -c  > /PATH/TO/DRIVE/backup_image.img.gz
  ```
  (adapt `mmcblk0` with your SD Card actual dev name)

## Restoring a backup
Same procedure but goes the other way round:
```bash
  gunzip  backup_image.img.gz
  sudo sh -c 'cat backup_image.img > /dev/mmcblk0'
```
(again, adapt `mmcblk0` with your SD Card actual dev name)
