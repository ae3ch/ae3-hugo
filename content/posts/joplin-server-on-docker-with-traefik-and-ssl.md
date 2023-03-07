---
title: "How to run Joplin Server on Docker with Traefik and SSL"
date: 2021-06-17 13:51:55+01:00
# aliases: ["/first"]
tags:
    - docker
    - traefik
    - joplin
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
    image: "https://oggen.ae3.ch/api/ae3?blogtitle=How%20to%20run%20Joplin%20Server%20on%20Docker%20with%20Traefik%20and%20SSL" # image path/url
    alt: "How to run Joplin Server on Docker with Traefik and SSL" # alt text
    caption: "How to run Joplin Server on Docker with Traefik and SSL" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---
Joplin Server is a self-hosted solution to synchronize your own notes, it supports multiple clients for Desktop and Mobile and comes with an optional end-to-end encryption. 

At the time of writing this article only clients newer than version 2.x work with the Joplin Server. This means you need to download the latest version of the desktop client from the GitHub repo. ~~For the mobile apps, there is currently no new version that supports syncing with the Joplin server, but those are coming.~~

**Update 17. Jun 2021:** Jolpin for [iOS](https://apps.apple.com/app/joplin/id1315599797) and [Android](https://play.google.com/store/apps/details?id=net.cozic.joplin) received an update. As of today, mobile apps can also synchronize with a self-hosted Jolpin Server.

## Install Joplin Server on Docker with Traefik

### Preparation

{{< notice info >}}
If you don't have docker installed yet, you can find instructions for [Ubuntu](/install-docker-on-ubuntu) or [Debian](/docker-install-debian-10). This Guide uses docker-compose to run Traefik, therefore its necessary to also install docker-compose. The two linked guides will help you to setup docker-compose on your own host. 
{{< /notice >}}

First we will create an extra folder for all Joplin Server data:
```bash
mkdir -p /opt/containers/joplin
```

### docker-compose.yml for Joplin Server

```bash
vim /opt/containers/joplin/docker-compose.yml
```

```yaml
version: '3'
services:
    joplin-db:
        image: postgres:13.1
        volumes:
            - ./data/postgres:/var/lib/postgresql/data
        restart: unless-stopped
        environment:
            - APP_PORT=22300
            - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
            - POSTGRES_USER=${POSTGRES_USER}
            - POSTGRES_DB=${POSTGRES_DATABASE}
        networks:
            - joplin
    joplin-app:
        image: joplin/server:latest
        depends_on:
            - joplin-db
        restart: unless-stopped
        environment:
            - APP_BASE_URL=${APP_BASE_URL}
            - DB_CLIENT=pg
            - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
            - POSTGRES_DATABASE=${POSTGRES_DATABASE}
            - POSTGRES_USER=${POSTGRES_USER}
            - POSTGRES_PORT=${POSTGRES_PORT}
            - POSTGRES_HOST=joplin-db
        networks:
            - proxy
            - joplin
        labels:
            - "traefik.enable=true"
            - "traefik.http.routers.joplin-app.rule=Host(`joplin.ae3.ch`)"
            - "traefik.http.routers.joplin-app.entrypoints=http"
            - "traefik.http.middlewares.joplin-app-https-redirect.redirectscheme.scheme=https"
            - "traefik.http.middlewares.joplin-app-https-header.headers.customrequestheaders.X-Forwarded-Proto = http"
            - "traefik.http.routers.joplin-app-secure.entrypoints=https"
            - "traefik.http.routers.joplin-app-secure.rule=Host(`joplin.ae3.ch`)"
            - "traefik.http.routers.joplin-app-secure.tls=true"
            - "traefik.http.routers.joplin-app-secure.tls.certresolver=http"
            - "traefik.http.routers.joplin-app-secure.service=joplin-app"
            - "traefik.http.services.joplin-app.loadbalancer.server.port=22300"
            - "traefik.http.services.joplin-app.loadbalancer.passhostheader=true"
            - "traefik.docker.network=proxy"
networks:
  joplin:
  proxy:
    external: true
```

Make sure that you have put your own domain in the labels. Otherwise Traefik will not be able to recognize your domain. 

{{< notice info >}}
These instructions assume that you have already set up Traefik on your host. If you don't have Traefik running yet, you can follow [these](/install-use-traefik-reverse-proxy-docker) instructions. 
{{< /notice >}}
Next, we create an environment file in the same folder. All variable data will be stored in this file, for example the SQL login data. 

```bash
vim /opt/containers/joplin/.env
```

```bash
APP_BASE_URL=https://joplin.ae3.ch/ # CHANGE HERE
POSTGRES_PASSWORD=GiVaJcumfCrDW@9uAo # USE A RANDOM PASSWORD HERE
POSTGRES_DATABASE=joplin
POSTGRES_USER=joplin
POSTGRES_PORT=5432
```

## Start Joplin Server with docker-compose

That's it. Now you can simply start Joplin Server on your host.

```bash
docker-compose -f /opt/containers/joplin/docker-compose.yml up d
```

Visit your domain (https://joplin.example.com/login) and login with the following default credentials. Make sure to change these after the first login!

**Username:** admin@localhost
**Password:** admin

## FAQ 

### Why can't I synchronize my desktop/mobile client with Joplin Server?
You will probably get the following error when you try to connect your client to the Joplin server.
```bash
App: 400: POST /api/files/root:/:/children : Not allowed: POST (CODE 400)
```
This error is due to your local client wich is outdated. You need to download the latest client version (min. version 2.x) from Joplin's [GitHub](https://github.com/laurent22/joplin/releases) repo, only with version 2 the sync will work. 