---
layout: post
series_title: "Self hosted Nextcloud 20+ on a Single Board Computer: a complete guide"
toc_title: "PART 3"
title:  "[3/5] Self hosted Nextcloud 20+"
date:   2021-03-23 12:30:00 +0200
tag: nextcloud
permalink: /self-hosted-nextcloud-on-sbc-complete-guide-part3
---

## See also
* [PART 1](/self-hosted-nextcloud-on-sbc-complete-guide-part1)
* [PART 2](/self-hosted-nextcloud-on-sbc-complete-guide-part2)
* PART 3 ← You are here 🙂
* PART 4
* [PART 5](/self-hosted-nextcloud-on-sbc-complete-guide-part5)

<p class="info">
  💾 This part deals with the installation of Nextcloud itself, now that we have a minimal install + a strong filesystem to support it. Expect it to last years with minimal maintenance (mainly major releases to major releases updates) as it should be. Set it and forget it! 🏋️
</p>

<p class="warning">
⚠️ Temporarily deactivate <code>overlayroot</code> first as we will need to write access to `/`: <a href="/self-hosted-nextcloud-on-sbc-complete-guide-part5#deactivate-read-only-root">PART 5: deactivate read only root</a>, otherwise your configuration will be lost at next reboot!
</p>

## Installing MariaDB
We need to install MariaDB before Nextcloud as the latter will ask to which DB we want it to connect to.

I assume we will connect and external hard drive and install MariaDB + Nextcloud on it (at least their data files). Assuming drive is mounted in `/media/data` hereafter.

```bash
sudo mkdir /media/data
sudo chown -R sheeva:sheeva /media/data
sudo mount -t ext4 /dev/disk/by-uuid/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX /media/data/

# For a permanent mount at boot time, edit /etc/fstab :
# UUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX	/media/data/	ext4	defaults	0	2
```

We can now install MariaDB, specifying that the data files will be in a specific folder in `/media/data`:

```bash
# Installing packages
sudo apt install mariadb-server mariadb-client

# Stopping the service
sudo service mysql stop

# Moving datadir directory to external hard drive
mkdir -p /media/data/mysql/datadir
sudo rsync -av /var/lib/mysql/ /media/data/mysql/datadir # rsync to keep files creation dates
sudo chgrp -R mysql /media/data/mysql

# Update datadir in config
# datadir = /media/data/mysql/datadir
sudo vim /etc/mysql/mariadb.conf.d/50-server.cnf # → update datadir line

# Add permissions for MariaDB to start sockets
# See: https://support.plesk.com/hc/en-us/articles/213918525-MySQL-does-not-start-Bind-on-unix-socket-Permission-denied
# at 2 places, group is 'root' instead of 'mysql'
sudo chgrp mysql /var/run/mysqld/
sudo chgrp mysql /media/data/mysql/datadir/mysql/

# Restart mariadb service
sudo service mariadb start

# Secure installation
sudo mysql_secure_installation # will ask questions

# Move the original config file so that
# we are sure we are not using it anymore
sudo mv /var/lib/mysql /var/lib/mysql-old
```

Make sure the service is started at boot:
```bash
sudo systemctl is-enabled mysql.service # outputs enabled/disabled
sudo systemctl disable mysql.service
sudo systemctl enable mariadb.service
```

To access MySQL command line interface, Debian packages configures unix_socket authentication by default, meaning `root` user can log in without password (same for `$USER` via `sudo`):

```bash
sudo mysql -u root # no need of '-p' option
```
This means it's impossible to connect to MariaDB's cli via `mysql -u root -p` command, the password option is ignored. One can see it as a warning is logged in MariaDB's logs:

```bash
sudo tail /var/log/mysql/error.log
# [...]
# 2018-10-27 18:23:31 3063459840 [Warning] 'user' entry 'root@localhost' has both a password and an authentication plugin specified. The password will be ignored.
# [...]
```
Source: [MariaDB Ubuntu root login](https://stackoverflow.com/questions/43439111/mariadb-warning-rootlocalhost-has-both-the-password-will-be-ignored#43439309)

Source: [Relocate Datadir folder](https://serverfault.com/q/872478)

## Installing Nextcloud

#### Creating Nextcloud user
```bash
sudo mysql -u root
MariaDB [(none)]> CREATE USER 'nextcloud_user'@'localhost';
# Query OK, 0 rows affected (0.00 sec)
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nextcloud.* To 'nextcloud_user'@'localhost' IDENTIFIED BY '$YOURPASSWORD';
# Query OK, 0 rows affected (0.00 sec)
MariaDB [(none)]> exit
```

#### Installing Nextcloud required packages
The procedure is quiet straightforward as written in [the doc](https://docs.nextcloud.com/server/17/admin_manual/installation/source_installation.html#prerequisites-label).

Needed packages:
```bash
# Essentials:
sudo apt install apache2 php7.3/stable php7.3-gd/stable php7.3-xml/stable php7.3-mbstring/stable php7.3-zip/stable php7.3-curl/stable php7.3-json/stable libxml2

# Specifics:
sudo apt install php7.3-mysql # MySQL / MariaDB Connector
sudo apt install php-apcu     # Local cache → https://docs.nextcloud.com/server/latest/admin_manual/configuration_server/caching_configuration.html

# Higly recommended
# PHP module bz2 (recommended, required for extraction of apps)
# PHP module intl (increases language translation performance and fixes sorting of non-ASCII characters)
# php-imagick otherwise NextCloud security check overview complains
sudo apt install php7.3-bz2/stable php7.3-intl/stable php-imagick/stable php-gmp/stable php-bcmath/stable ssl-cert
 ```

⚠ Don't forget to configure cache in nextcloud's config ([APCU cache Doc](https://docs.nextcloud.com/server/12/admin_manual/configuration_server/caching_configuration.html#id1)), see below.
 
⚠ Pas besoin d'installer de module `mod_webdav` car NextCloud inclus un server webdav lui-même (voir le dernier point de [cette section](https://docs.nextcloud.com/server/12/admin_manual/installation/source_installation.html#prerequisites-for-manual-installation)).
 
⚠ No need to install `mod_webdav` module as Nextcloud now includes a webdav server itself (see last paragraph of [this section](https://docs.nextcloud.com/server/latest/admin_manual/installation/source_installation.html#prerequisites-for-manual-installation)).


Source: [Nextcloud command line installation](https://docs.nextcloud.com/server/latest/admin_manual/configuration_server/occ_command.html#command-line-installation-label)

#### Creating a Nextcloud data folder
Folder must be writable to user `www-data`.

```bash
mkdir /media/data/nextcloud /media/data/nextcloud_tmp_dir
sudo chmod 0770 /media/data/nextcloud /media/data/nextcloud_tmp_dir
sudo apt install acl
setfacl -m u:www-data:rwx /media/data/nextcloud
setfacl -m u:www-data:rwx /media/data/nextcloud_tmp_dir
```

#### Downloading latest Nextcloud archive
There is unfortunately no official PPA for Debian, one must download a `.zip` archive from [Nextcloud homepage](https://nextcloud.com/install/#). Upgrades are done via a Nextcloud app ([Updater App](https://docs.nextcloud.com/server/latest/admin_manual/maintenance/upgrade.html)).
 
 ```bash
 cd /tmp
 
 # NextCloud
 curl -L https://download.nextcloud.com/server/releases/nextcloud-21.0.0.zip -o nextcloud-21.0.0.zip
 
 # SHA256
 curl -L https://download.nextcloud.com/server/releases/nextcloud-21.0.0.zip.sha256 -o nextcloud-21.0.0.sha256
 
 # Check download OK :
 sha256sum -c nextcloud-21.0.0.sha256 < nextcloud-21.0.0.zip
 # nextcloud-21.0.0.zip: OK
 ```

#### Launching Nextcloud installer
```bash
unzip nextcloud-21.0.0.zip
sudo mv nextcloud /var/www/
sudo chown -R www-data:www-data /var/www/nextcloud
sudo chmod 0770 /var/www/nextcloud
sudo chmod 0770 /media/data/nextcloud/
# Ensuite: https://docs.nextcloud.com/server/latest/admin_manual/installation/command_line_installation.html

# Launch Nextcloud install script via the following command :
# Adapt $DATABASE_PWD, $ADMIN_LOGIN and $ADMIN_PWD accordingly
su root
cd /var/www/nextcloud
sudo -u www-data php occ maintenance:install --database "mysql" --database-name "nextcloud" --database-user "nextcloud_user" --database-pass "$DATABASE_PWD" --admin-user "$ADMIN_LOGIN" --admin-pass "$ADMIN_PWD" --data-dir "/media/data/nextcloud"
exit
# Nextcloud was successfully installed

# If you encounter the following error:
# "Username is invalid because files already exist for this user"
# Problem is documented here: https://github.com/nextcloud/server/blob/master/lib/private/User/Manager.php#L630
# You unfortunately need to delete /media/data/nextcloud/ 's content, otherwise installer find some files and considers admin user already exist. There is probably a cleaner way to do this but I assume this is a fresh install. 😇
```

Yeah 😎!