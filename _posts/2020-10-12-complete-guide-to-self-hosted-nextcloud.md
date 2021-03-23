---
layout: post
title:  "[1/5] Self hosted Nextcloud 20+"
date:   2020-10-12 12:30:00 +0200
categories: nextcloud embedded linux
permalink: /self-hosted-nextcloud-on-sbc-complete-guide-part1
last_update: 2021-03-23 12:30:00 +0200
---

Last update: **{{ page.last_update | date: "%B %d, %Y %H:%m" }}**


<p class="info">
  📚 Installation on a single board computer: a complete guide
</p>

## Introduction
In this long post, I'll describe how to configure a single board computer such as the **Raspberry Pi** or the **Asus Tinker Board** from A to Z to support a self-hosted Nextcloud 20+ instance. We'll go through the following steps:

* PART 1:
  * [Installing Official image](#installing-official-image)
  * [Cleaning up Official image](#cleaning-up-official-image) (becoming a minimal installation)
  * [Securing Official image ](#securing-official-image)(removing clunky pre-installed configs to strengthen the installation)
* [PART 2](/self-hosted-nextcloud-on-sbc-complete-guide-part2):
  * [Partitioning a read-only root](/self-hosted-nextcloud-on-sbc-complete-guide-part2#read-only-root) (via `overlayroot`)
  * [*(Optional)* installing a self-compiled kernel from kernel.org](/self-hosted-nextcloud-on-sbc-complete-guide-part2) (😎!)
* PART 3:
  * [Installing + configuring MariaDB]()
  * [Installing Nextcloud pre-requisites]()
  * [Installing Nextcloud 20+]()
  * [Configuring Nginx or Apache2 in front of it]()
  * [Tuning and tweaking Nextcloud 20+ freshly installed]()
* Part 4:
  * [Installing Let's Encrypt SSL wildcards certificates]()
  * [Setting up Dynhost via `ddclient`]()
* Part 5:
  * [Bonus: updating Nextcloud despite read-only root]()
  * [Bonus: making incremental `rsync` backups]()
  * [**Bonus: Backup your SD Card**]() (do that regularly while going through this guide)

My personal board is an Asus Tinker Board, but I've played with Raspberry Pi 4s as well before, and the procedure is roughly the same (>90%). I'll describe the differences when they arise (mainly at the image installation and cleanup steps).

## Installing official image

#### Raspberry pi
Download [Raspberry Pi OS (32-bit) Lite](https://www.raspberrypi.org/downloads/raspberry-pi-os/) image.

Unzip the file; you'll end up with an .img file like `2020-08-20-raspios-buster-armhf-lite.img`. The image is a dump of a 2Gb partition with ≃ 435Mb of content. We'll reduce that shortly.

#### Tinker board
Download the latest image [here](https://www.asus.com/uk/Single-Board-Computer/Tinker-Board/HelpDesk_Download/) (**2.1.16** at this time of writing). It weights ≃ 2Gb. We'll shortly remove spurious content.

Unzip the file; you'll end up with an .img file like `Tinker_Board-Debian-Stretch-V2.1.16-20200813.img`. The image is a dump of a 2Gb partition with a lot of content to start tinkering. We'll reduce that shortly.

#### Putting the image on the SD Card

To transfer the `.img` content from your hard drive to the SDCard, copy its content via the `dd` command (Linux). Be aware you have to know which device name your SD card receives from the OS, **your mileage may (will) vary**. The best way to know is probably to `ls /dev` folder, then insert the SD Card, and `ls` it again. The newly added device is your SD Card (e.g. `/dev/mmcblk0`). If your SD Card has multiple partition, they will appear with as suffix like `pX` (e.g: `/dev/mmcblk0p1`, `/dev/mmcblk0p2`, ...). We are only interested in the device name (e.g. `/dev/mmcblk0`), **not** the partition name(s).

```bash
# Transfer the .img content to SD Card
sudo sh -c 'cat Tinker_Board-Debian-Stretch-V2.1.16-20200813.img > /dev/mmcblk0'
```
⌛ This will take some time, be patient. 

Once done, starting `gparted` can be handy to extend the copied partition. SD Cards have > than 16Gb by now, and the copied image is a 2Gb partition.

## Cleaning up official image

#### First Boot
🥳 Hooray, your SD Card is ready to go! Let's (u-)boot now! 🎉

<p class="warning">
  You'll need a solid Power Supply, preferably 5V 3A. 2A isn't enough to power an external hard drive that has no external power supply. You'll notice some shortages and weird behaviors if your power supply isn't powerful enough. I've experienced this with <strong>both</strong> Raspberry Pi and TinkerBoard!
</p>

```bash
# Connect through SSH:

# Raspberry pi
ssh pi@192.168.1.49 # Default Password: raspberry

# Tinker Board
ssh linaro@192.168.1.49 # Default Password: linaro
```

These default users have been created to start tinkering out of the box and MUST be removed because they have too many privileges (like `sudo su -` for root access without password!). Prefer creating a new user at first boot and remove them later; once we'll have removed all GUI (automatic login can be confused if the default user doesn't exist anymore).

##### Creating new user
```bash
sudo su - # otherwise useradd is not in the PATH
useradd -m "$MYUSER"
 echo "$MYUSER:$MYPASSWORD" | chpasswd
adduser "$MYUSER" sudo # adding myuser to sudo
chsh -s /bin/bash "$MYUSER" # default shell
```
🧐 Pro Tip: notice the space character before `echo`. This line won't be logged in your prompt history!

##### tinker-config / raspi-config
Either raspberry pi and tinker board have a script to facilitate their configuration: boot type (CLI or GUI), locale, timezone, ... very useful.

```bash
# Raspberry pi
sudo raspi-config

# Tinker Board
sudo tinker-config
```

##### Static Network
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

##### Removing ALL GUI (Desktop, program,...)

```bash
sudo tinker-config # configure boot not to start Desktop
sudo apt-get purge --auto-remove libx11-6 libwayland-client0 x11-common libwayland-server0
```
This is VERY EFFECTIVE! [source](https://unix.stackexchange.com/questions/424969/how-can-i-remove-all-packages-related-to-gui-in-debian)

*NB: I could remove >769Mb with this method and boot time considerably decreased!*

You now have a minimal install 😎.

## Securing official image

##### Updating root password
```bash
sudo su -
# method 1 :
passwd
# method 2 :
 (echo "$ROOT_PWD"; echo "$ROOT_PWD") | passwd root
```

##### Securing SSH
Only allow connection via `$MYUSER`.

```bash
sudo vim /etc/ssh/sshd_config

# Add/Modify the following directives:
AllowUsers MYUSER
PermitRootLogin no
```
Other ssh tweaks can be found [here](https://www.digitalocean.com/community/tutorials/ssh-essentials-working-with-ssh-servers-clients-and-keys).


## Next

See [PART 2](/self-hosted-nextcloud-on-sbc-complete-guide-part2)