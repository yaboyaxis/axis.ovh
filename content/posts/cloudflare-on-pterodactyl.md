---
title: "Configuring Cloudflare on Pterodactyl"
date: 2021-04-29T02:58:58
draft: false
tags:
  - cloudflare
  - pterodactyl
  - networking
---

## Preface

[Pterodactyl](https://pterodactyl.io) is a game panel meant to make hosting a minecraft (or really any) server quite easy! I personally love the Pterodactyl project a lot, and wanted to find a way to proxy it on cloudflare, as well as run Pterodactyl without any errors.

The main benefit to being able to proxy Pterodactyl is the benefit of not showing your origin IP, even though this wouldn't be useful for if Wings and Panel are on the same server, Cloudflare improves speed and security.

## What's the issue with Cloudflare and Pterodactyl?

Pterodactyl and Cloudflare (by default) don't mix very well together. Before configuring it correctly, I got [521](https://community.cloudflare.com/t/community-tip-fixing-error-521-web-server-is-down/42461) and [525](https://community.cloudflare.com/t/community-tip-fixing-error-525-ssl-handshake-failed/44256) errors while configuring Wings and the Panel itself.

There's some ways we can get around it, which involves changing some settings in Cloudflare and in the Wings config.

## Cloudflare Configuration

Firstly, we have to change our Cloudflare SSL mode to **Full**, which will allow Cloudflare to verify our certificate we'll put on the server later.

![Enable Cloudflare Full SSL](https://i.imgur.com/AdmyYST.png)

After that, we'll generate an **Origin Certificate**. This allows Cloudflare to verify the `cloudflare -> server` hop (whereas Flexible only verifies the `web -> cloudflare` hop).

![Generate Origin Certificate 1](https://i.imgur.com/iXRbUR6.png)

You can generate the certificate using these settings. Put the domain as **your panel domain**.
After you've generated the certificate, copy the certificate and key to a `.pem` file on your server. Make sure to store it in a location you remember. You'll then see the below menu.

![Generate Origin Certificate 2](https://i.imgur.com/CqXEWk0.png)

After you've generated the certificate, please continue.

## Panel and Nginx Configuration

There is a small part of the panel `.env` we have to change. Usually this is located in `/var/www/pterodactyl`, but it may be different for you.

`.env` files are hidden, so you can do `cat .env` to view it, and then use a text editor to edit it. At **the bottom**, add this value:

```
TRUSTED_PROXIES=103.21.244.0/22,103.22.200.0/22,103.31.4.0/22,104.16.0.0/12,108.162.192.0/18,131.0.72.0/22,141.101.64.0/18,162.158.0.0/15,172.64.0.0/13,173.245.48.0/20,188.114.96.0/20,190.93.240.0/20,197.234.240.0/22,198.41.128.0/17
```

Above will tell Pterodactyl to accept connections coming from Cloudflare.

If you're not using Nginx for this step, you should be able to skip this. Go to the `pterodactyl.conf` file in the nginx `sites-enabled` folder, and add this to the `location / {}` block:

```nginx
location / {
      try_files $uri $uri/ /index.php?$query_string;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header Host $host;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
      proxy_redirect off;
      proxy_buffering off;
      proxy_request_buffering off;
}
```

Above will also tell Nginx to properly proxy the connections coming from Cloudflare.

## Wings Configuration

In the node configuration on the panel, you'll have the node you'll want to configure, please go to the `Settings` section of the page, and find the `General Configuration` tab, like shown below.

![Node Configuration](https://i.imgur.com/F1aKipZ.png)

Change the **Daemon Port** to ANY cloudflare compatible **HTTPS** port [here](https://support.cloudflare.com/hc/en-us/articles/200169156-Identifying-network-ports-compatible-with-Cloudflare-s-proxy).

After saving that, go to the config location above, and edit these lines in the `api` section:

```yml
api:
  ssl:
    enabled: true
    cert: /path/to/ssl/cert.pem
    key: /path/to/ssl/key.pem
    # Rest of the settings below can stay the same
```

Save the file, and run `systemctl restart wings` (or `wings` if not enabled on systemd).

## Ending

After that, Pterodactyl should be fully able to accept incoming Cloudflare connections. The benefits of proxying Pterodactyl are tremendous, and it doesn't require that much more configuration!
