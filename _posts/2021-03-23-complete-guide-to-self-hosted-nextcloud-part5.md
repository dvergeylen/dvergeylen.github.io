---
layout: post
title:  "[5/5] Self hosted Nextcloud 20+"
date:   2021-03-23 12:30:00 +0200
categories: nextcloud embedded linux
permalink: /self-hosted-nextcloud-on-sbc-complete-guide-part5
---

## See also
* [PART 1](/self-hosted-nextcloud-on-sbc-complete-guide-part1)
* [PART 2](/self-hosted-nextcloud-on-sbc-complete-guide-part2)
* [PART 3](/self-hosted-nextcloud-on-sbc-complete-guide-part3)
* PART 4
* PART 5 ← You are here 🙂

<p class="info">
  🏗️ This part deals with maintenance burden. Not the funniest part but always needed to ensure a healthy system. I've divided the sections as if they were 'flight rules', they aren't meant to be read sequentially.
</p>

## Deactivate read only root
When updates are needed or simply updating the configuration is needed, one could need to temporarily deactivate the read only filesystem.

Although `overlayroot` package gives an `overlayroot-chroot` command, don't use it for two reasons:

1. The `/media/root-ro` directory is remounted as `ro` and changes wouldn't persist
2. Packages updates / installation might break because some operations are forbidden in chrooted environment. You might experience some `Running in chroot, ignoring request.` errors.

```bash
sudo mount -o remount,defaults,noatime,rw -t ext4 /dev/mmcblk0p2 /media/root-ro
sudo vim /media/root-ro/etc/overlayroot.local.conf
# → Add '#' before `overlayroot="tmpfs:swap=1,recurse=0"` line
sudo reboot

# One liner:
sudo sed -i s/overlayroot/#overlayroot/g /media/root-ro/etc/overlayroot.local.conf
```