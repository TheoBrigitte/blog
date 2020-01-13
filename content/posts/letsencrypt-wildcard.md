---
title: "Let's Encrypt wildcard certificates"
date: 2020-01-12T23:11:58+01:00
description: "Setup wildcard certificate with let's encrypt and certbot"
---

## Install certbot

Certbot is the official command line tool to manage Let's Encrypt certificates.

Since certbot is a python application, it is best to install it via python packet manager: pip.

```
# pip install certbot

```

In order to issue wildcard certificates you need to use DNS authentication method. Certbot has many DNS providers covered, you can list them with: `certbot plugins`

I will continue using OVH as DNS provider.

```
# pip install certbot-dns-ovh
```

## Create an account

Before being able to issue certificates, you need to create an account, certbot has you covered.

```
# certbot register
```

## Issue wildcard certificate

First check how to use your dns plugin

```
# certbot -h dns-ovh
usage: 
  certbot [SUBCOMMAND] [options] [-d DOMAIN] [-d DOMAIN] ...

Certbot can obtain and install HTTPS/TLS/SSL certificates.  By default,
it will attempt to use a webserver both for obtaining and installing the
certificate. 

optional arguments:
  -h, --help            show this help message and exit
  -c CONFIG_FILE, --config CONFIG_FILE
                        path to config file (default: /etc/letsencrypt/cli.ini
                        and ~/.config/letsencrypt/cli.ini)

dns-ovh:
  Obtain certificates using a DNS TXT record (if you are using OVH for DNS).

  --dns-ovh-propagation-seconds DNS_OVH_PROPAGATION_SECONDS
                        The number of seconds to wait for DNS to propagate
                        before asking the ACME server to verify the DNS
                        record. (default: 30)
  --dns-ovh-credentials DNS_OVH_CREDENTIALS
                        OVH credentials INI file. (default: None)
```

According to documentation https://certbot.eff.org/docs/using.html#dns-plugins, I need to create a file with my OVH credential in the following format:

```
# OVH API credentials used by Certbot
dns_ovh_endpoint = ovh-eu
dns_ovh_application_key = MDAwMDAwMDAwMDAw
dns_ovh_application_secret = MDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAw
dns_ovh_consumer_key = MDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAw
```

Now I can generate my certificate by using:

```
# certbot certonly --dns-ovh --dns-ovh-credentials ./ovh_credentials.conf -d 'theobrigitte.com' -d '*.theobrigitte.com' 
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator dns-ovh, Installer None
Obtaining a new certificate
Performing the following challenges:
dns-01 challenge for theobrigitte.com
dns-01 challenge for theobrigitte.com
Waiting 30 seconds for DNS changes to propagate
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/theobrigitte.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/theobrigitte.com/privkey.pem
   Your cert will expire on 2020-01-12. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

Happy encryption :)
