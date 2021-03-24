---
layout: post
series_title: "Self hosted Nextcloud 20+ on a Single Board Computer: a complete guide"
toc_title: "PART 4"
title:  "[4/5] Self hosted Nextcloud 20+"
date:   2021-03-23 12:30:00 +0200
tag: nextcloud
permalink: /self-hosted-nextcloud-on-sbc-complete-guide-part4
---

## See also
* [PART 1](/self-hosted-nextcloud-on-sbc-complete-guide-part1)
* [PART 2](/self-hosted-nextcloud-on-sbc-complete-guide-part2)
* [PART 3](/self-hosted-nextcloud-on-sbc-complete-guide-part3)
* PART 4 ← You are here 🙂
* [PART 5](/self-hosted-nextcloud-on-sbc-complete-guide-part5)

<p class="info">
  🎯 This part deals with TLS certificate generation (wildcard) and DNS client configuration. You are next to the end!
</p>

## Let's Encrypt TLS Certificate
Your server serves requests à port `443`, which is reserved port for `https`. It must serve a valid TLS certificate, signed by a third party. Let's Encrypt is a non profit issuing TLS Certificates for free, which are perfectly valid. At this time of writing, they issue >1.5M certificates per day!

They conceived an implemented a new protocoel to allow certificate issuance in an automated way. Its second version, ACMEv2, support wildcard certificate types (`*.mydomain.com`). Neat!

Various clients appeared on the web to implement the ACME protocol. I tried a few of them but I have to say [acme.sh](https://github.com/acmesh-official/acme.sh) is my favourite.

#### Generate SSL/TLS Certificate
In order to issue wildcard certificates, `acme.sh` needs to be able to write some DNS records to proove it has sufficient permissions to administer the domain name (and its subdomains). This requires an application key + secret from your DNS provider. The procedure diverges from provider to provider but `acme.sh` support [the main ones]( https://github.com/acmesh-official/acme.sh/wiki/dnsapi), given the credentials.

I will use OVH as an example because its the one I use and its the #1 in Europe. 

```bash
# Git clone du logiciel
git clone https://github.com/acmesh-official/acme.sh.git
cd acme.sh

# Get your application + secret keys from your DNS provider.
# I use OVH (leader in Europe), where you can fetch them here:
# https://eu.api.ovh.com/createApp/

# Export OVH Application Key
 export OVH_AK="XXX"

# Export OVH Application Secret
 export OVH_AS="YYY"

# Start acme.sh with the requested domain name (+ wildcard)
# See https://github.com/acmesh-official/acme.sh/wiki/dnsapi
# for per DNS provider instructions
./acme.sh --issue -d my-domain.com -d '*.my-domain.com' --dns dns_ovh

# acme outputs the files location. Copy the private key path (*.key) and the fullchain path (fullcain.cer)
# copier les originaux vers sheeva de nextcloud
# /home/$USER/.acme.sh/my-domain.com/my-domain.com.key
# /home/$USER/.acme.sh/my-domain.com/fullchain.cer
```

#### Generate a custom DHParams file
```bash
openssl dhparam -out /etc/nginx/dhparam.pem 4096
```

#### update Nginx / Apache configuration
You can now edit the Nginx / Apache configuration from part 3 with the newly created files:

* Nginx:
  
  ```conf
  # TLS CONFIGURATION
  ssl_certificate /path/to/fullchain.cer;
  ssl_certificate_key /path/to/private_key.key;
  ssl_dhparam /path/to/dhparam.pem;
  ```
* Apache:

  ```conf
  # TLS CONFIGURATION
  SSLEngine on
  SSLCertificateFile /path/to/fullchain.cer
  SSLCertificateKeyFile /path/to/private_key.key
  SSLOpenSSLConfCmd DHParameters /path/to/dhparam.pem
  ```

Restart your server and you are good to go: `sudo systemctl restart nginx.service`

## Dynamic host via ddclient
`ddclient` is a client that updates IPs for dynamic hosts and supports multiple DNS providers. It runs as a `systemd` daemon so no need for CRON stuff whatsoever!

```bash
sudo apt install ddclient

# The install script will ask questions, here are my answers for OVH DNS provider (in FR 🤷)
#    Fournisseur de service DNS : Other
#    Serveur de DNS dynamique : www.ovh.com
#    Protocole de mise à jour : dyndns2
#    Identifiant pour le service : your-ovh-identifier
#    Mot de passe : your password
#    Interface réseau utilisé par le service de DNS : eth0
#    Nom de domaine de DNS dynamique : your-domain-name.com
```

It will already work as is, but it will use the local IP as dynamic resolution. One must update the file `/etc/ddclient.conf` as follow:
```bash
# Configuration file for ddclient generated by debconf
#
# /etc/ddclient.conf

daemon=5m,                                                         \
protocol=dyndns2,                                                  \
use=web, web=checkip.dyndns.com/, web-skip='Current IP Address: ', \
server=www.ovh.com,                                                \
login=your-ovh-identifier,                                       \
password='wildcard',                                               \
you-domain-names.com,comma-separated
```

`ddclient`'s manapage gives some useful examples: `/usr/sbin/ddclient --help`
[Source [FR]](https://perhonen.fr/blog/2016/03/dynhost-dyndns-de-chez-ovh-2446)

## Conclusion
If you followed until this section you should now have a pretty solid Nextcloud installation. There is of course always room for improvements and I would be thrilled to hear your suggestions! You can always contact me, there are a couple of ways to reach me out listed on the homepage. The Github repo of this blog is [open for discussions](https://github.com/dvergeylen/dvergeylen.github.io/discussions), so don't hesitate to start one there if you want to discuss about specific topics. Thanks and all the best! 🥂

## Next
There is a 5th 'Bonus' part to this blog post serie, namely about maintenance tasks, check it out [here](/self-hosted-nextcloud-on-sbc-complete-guide-part5).