---
title: "AdGuard Home with Docker, DNS-over-HTTPS and Traefik"
date: 2022-06-02 17:19:17+01:00
# aliases: ["/first"]
tags:
    - docker
    - traefik
    - dns
author: "ae3"
draft: false
hidemeta: false
comments: false
description: ""
#canonicalURL: "https://canonical.url/to/page"
hideSummary: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "https://oggen.ae3.ch/api/ae3?blogtitle=AdGuard%20Home%20with%20Docker,%20DNS-over-HTTPS%20and%20Traefik" # image path/url
    alt: "AdGuard Home with Docker, DNS-over-HTTPS and Traefik" # alt text
    caption: "AdGuard Home with Docker, DNS-over-HTTPS and Traefik" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---
AdGuard Home is a Selfhosted DNS server that can block Ads and Malware Domains inside your network. AdGuard Home is an alternative to a PiHole, with one big advantage: AdGuard can natively do DNS-over-TLS and DNS-over-HTTPS, and expirmentell it even provides support for DNS-over-QUIC. With a PiHole this would theoretically be possible as well, but you need additional software and some manual configuration.

For this reason, today I want to show you how to set up AdGuard with Docker. We will use Traefik as a reverse proxy and focus on DNS-over-HTTPS.  

## AdGuard Home with Docker, DNS-over-HTTPS and Traefik
### Preparation

{{< notice info >}}
If you don't have docker installed yet, you can find instructions for [Ubuntu](/install-docker-on-ubuntu) or [Debian](/docker-install-debian-10). This Guide uses docker compose to run Traefik, therefore its necessary to install the latest docker version available.

Please also make sure that you already have Traefik installed on your server. If you haven't done that yet, you can find a [step by step guide here](https://ae3.ch/install-use-traefik-reverse-proxy-docker). 
{{< /notice >}}

![adguard-home-docker-traefik](/images/adguard-home-docker-traefik.png "adguard-home-docker-traefik")

AdGuard Home can be run both on your own network and on a remote server. 
This tutorial is specifically about the installation on a server that is outside of your own network. 

We will therefore limit ourselves to DNS-over-HTTPS, as this means that our DNS queries between the computer and the external server are encrypted at all times. This way your ISP or someone in the network can't find out which DNS requests are coming from your device. 

Ok let's start.

Frist we create a few files and folders to work with. 

```bash
mkdir -p /opt/containers/adguard
mkdir -p /opt/containers/adguard/conf
mkdir -p /opt/containers/adguard/work
```

### AdGuard Home docker-compose.yml
Next up we are creating our docker-compose.yml for AdGuard
```bash
vim /opt/containers/adguard/docker-compose.yml
```

```bash
version: '3'
services:
  adguard:
    image: adguard/adguardhome:latest
    container_name: adguard
    restart: unless-stopped
    networks:
      - proxy
    volumes:
      - /opt/containers/adguard/work:/opt/adguardhome/work
      - /opt/containers/adguard/conf:/opt/adguardhome/conf
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.adguard.entrypoints=http"
      - "traefik.http.routers.adguard.rule=Host(`dns.ae3.ch`)" # change with your own domain/sub domain
      - "traefik.http.middlewares.adguard-https-redirect.redirectscheme.scheme=https"
      - "traefik.http.routers.adguard.middlewares=adguard-https-redirect"
      - "traefik.http.routers.adguard-secure.entrypoints=https"
      - "traefik.http.routers.adguard-secure.rule=Host(`dns.ae3.ch`)" # change with your own domain/sub domain
      - "traefik.http.routers.adguard-secure.tls=true"
      - "traefik.http.routers.adguard-secure.tls.certresolver=http"
      - "traefik.http.routers.adguard-secure.service=adguard"
      - "traefik.http.services.adguard.loadbalancer.server.port=3000"
      - "traefik.docker.network=proxy"

networks:
  proxy:
    external: true
```
At first we need traefik to proxy port 3000, after the initial setup of AdGuard Home we need to change the port to 80. 


## Run our AdGuard Home container
```bash
cd /opt/containers/adguard
docker compose up -d
```

You can now visit `https://yourdomain.tld` to finish the initial setup of AdGuard Home. 
Just follow the steps and create a admin user for later.

Once you're done, we'll stop the container again and take care of the final adjustments.

```bash
docker compose down
```

## AdGuard Home DNS-over-HTTPS Settings
AdGuard Home takes care of the encrypted HTTPS connection itself in the default settings. However, since we use Traefik and thus save ourselves the manual renewal of the SSL certificate (Traefik does this for us completely automatically), we now have to make a few adjustments to the settings. 

To do this, we go to the config directory of AdGuard Home. 
```bash
cd /opt/containers/adguard/conf
vim AdGuardHome.yaml
```

In the "tls:" section we now need to enable the TLS connection and write our domain in the config with which we want to use AdGuard Home. We also need to change `allow_unencrypted_doh: false` to `true`. This section should now look like this:

```bash
tls:
  enabled: true
  server_name: yourdomain.tld # change here
  force_https: false
  port_https: 443
  port_dns_over_tls: 853
  port_dns_over_quic: 784
  port_dnscrypt: 0
  dnscrypt_config_file: ""
  allow_unencrypted_doh: true # this setting is important, since we are using a reverse proxy to access DoH
  strict_sni_check: false
  certificate_chain: ""
  private_key: ""
  certificate_path: ""
  private_key_path: ""
```

Now save and close the config file. We now edit our docker-compose.yml one last time.

Since the initial setup is already complete, we now need to change the port from 3000 to 80, which Traefik should use to connect to our container. 

```bash
# change this line
      - "traefik.http.services.adguard.loadbalancer.server.port=3000"
      
# to this
      - "traefik.http.services.adguard.loadbalancer.server.port=80"
```

Now we are done and can save the file and start our container one last time. 
```bash
cd /opt/containers/adguard
docker compose up -d
```

## Start using AdGuard Home with your devices and DNS-over-HTTPS
If you now open your domain in the browser you can log in to the backend of AdGuard Home and now link all your devices to AdGuard Home and block annoying ads or dangerous malware sites. 

Since we use AdGuard on an external server, we don't use the usual DNS port 53 like we would do with public DNS servers like 1.1.1.1. 
No we use DNS-over-HTTPS, so your DNS queries are transmitted over an HTTPS connection encrypted on the Internet. 

To use your devices with AdGuard and DNS-over-HTTPS, the easiest way is to click on the "Setup Guide" menu item in the AdGuard Dashboard.

![adguard-home-setup](/images/adguard-home-setup.png "adguard-home-setup")

Your DNS-over-HTTPS address should look like this:
`https://yourdomain.tld/dns-quer/device-name` 

Firefox and Google Chrome can be set to directly use the DoH address for the DNS query. 
[https://support.mozilla.org/en-US/kb/firefox-dns-over-https](https://support.mozilla.org/en-US/kb/firefox-dns-over-https)