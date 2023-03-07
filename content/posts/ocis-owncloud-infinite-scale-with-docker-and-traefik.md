---
title: "oCIS - ownCloud Infinite Scale with Docker and Traefik"
date: 2022-06-09 14:51:52+01:00
# aliases: ["/first"]
tags:
    - docker
    - traefik
    - ocis
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
    image: "https://oggen.ae3.ch/api/ae3?blogtitle=ownCloud%20Infinite%20Scale%20with%20Docker%20and%20Traefik" # image path/url
    alt: "oCIS - ownCloud Infinite Scale with Docker and Traefik" # alt text
    caption: "oCIS - ownCloud Infinite Scale with Docker and Traefik" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---
OwnCloud decided to develop a completely new product a few months ago. In the meantime, there are already the first beta versions of the new OwnCloud with the name oCIS. oCIS stands for ownCloud Infinite Scale and should make clear that this cloud solution should be very scalable. 

Newly, it is sufficient to start a single container with oCIS, which already contains all the necessary dependencies. So no database and no further containers are necessary to self-host oCIS. 

In the following I would like to show you a setup how I run oCIS with Traefik as reverse proxy on my own server since the release of the first tech previews. 

# oCIS - ownCloud Infinite Scale with Docker and Traefik
## Preparation

{{< notice info >}}
If you don't have docker installed yet, you can find instructions for [Ubuntu](/install-docker-on-ubuntu) or [Debian](/docker-install-debian-10). This Guide uses docker compose to run Traefik, therefore its necessary to install the latest docker version available.

Please also make sure that you already have Traefik installed on your server. If you haven't done that yet, you can find a [step by step guide here](https://ae3.ch/install-use-traefik-reverse-proxy-docker). 
{{< /notice >}}

In this tutorial we assume that you have already started traefik. 

## oCIS environment config
```bash
mkdir -p /opt/containers/ocis
vim /opt/containers/ocis/.env
```

```bash
# Setting to allow non-https traffic between traefik and ocis
INSECURE=true
### oCIS settings ###
# oCIS version. Defaults to "latest"
OCIS_DOCKER_TAG=
# Domain of oCIS, where you can find the frontend. Defaults to "ocis.owncloud.test"
OCIS_DOMAIN=ocis.yourdomain.tld
# oCIS admin user password. Defaults to "admin".
ADMIN_PASSWORD=klPPHGEPIaudZmhDgT6JiZlJv6lzmtpXwNuNyNBDtg
# The demo users should not be created on a production instance
# because their passwords are public. Defaults to "false".
DEMO_USERS=false
```

`OCIS-DOMAIN` and `ADMIN_PASSWORD` should be changed with your own domain and password. 

## docker-compose.yml for oCIS

```bash
vim /opt/containers/ocis/docker-compose.yml
```

```bash
---
version: "3.7"

services:
  ocis:
    image: owncloud/ocis:${OCIS_DOCKER_TAG:-latest}
    user: root
    networks:
      proxy:
    entrypoint:
      - /bin/sh
    command: ["-c", "ocis init || true; ocis server"]
    environment:
      OCIS_URL: https://${OCIS_DOMAIN:-ocis.owncloud.test}
      OCIS_LOG_LEVEL: ${OCIS_LOG_LEVEL:-error} # make oCIS less verbose
      PROXY_TLS: "false" # do not use SSL between Traefik and oCIS
      OCIS_INSECURE: "${INSECURE:-false}"
      # basic auth (not recommended, but needed for eg. WebDav clients that do not support OpenID Connect)
      PROXY_ENABLE_BASIC_AUTH: "${PROXY_ENABLE_BASIC_AUTH:-false}"
      # admin user password
      IDM_ADMIN_PASSWORD: "${ADMIN_PASSWORD:-admin}" # this overrides the admin password from the configuration file
      # demo users
      IDM_CREATE_DEMO_USERS: "${DEMO_USERS:-false}"
    volumes:
      - /opt/containers/ocis/ocis-config:/etc/ocis
      - /opt/containers/ocis/ocis-data:/var/lib/ocis
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.ocis.entrypoints=https"
      - "traefik.http.routers.ocis.rule=Host(`${OCIS_DOMAIN:-ocis.owncloud.test}`)"
      - "traefik.http.routers.ocis.tls.certresolver=http"
      - "traefik.http.routers.ocis.service=ocis"
      - "traefik.http.services.ocis.loadbalancer.server.port=9200"
    logging:
      driver: "local"
    restart: always

volumes:
  ocis-config:
  ocis-data:

networks:
  proxy:
    external: true
```

## Run oCIS with our config
```bash
cd /opt/containers/ocis/
docker compose up -d
```
![ocis-docker-traefik-setup](/images/ocis-docker-traefik-setup.png "ocis-docker-traefik-setup")

After a few seconds you will be able to access your fresh oCIS Cloud with your domain name (ex. ocis.yourdomain.tld). Please make sure to access your site with `https://` otherwise traefik will show a 404 error. 

For the login you just use the user `admin` and the password we defined in the `.env` with `ADMIN_PASSWORD`. 