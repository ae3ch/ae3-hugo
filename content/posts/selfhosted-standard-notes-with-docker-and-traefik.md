---
title: "Selfhosted Standard Notes with Docker and Traefik"
date: 2021-07-12 15:36:44+01:00
# aliases: ["/first"]
tags:
    - docker
    - traefik
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
    image: "https://oggen.ae3.ch/api/ae3?blogtitle=Selfhosted%20Standard%20Notes%20with%20Docker%20and%20Traefik" # image path/url
    alt: "Selfhosted Standard Notes with Docker and Traefik" # alt text
    caption: "Selfhosted Standard Notes with Docker and Traefik" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---
Standard Notes is a note-taking app that you can use on all major platforms and mobile devices. 
With Standard Notes, your notes are end-to-end encrypted and therefore protected from prying eyes. 

I've been a fan of Standard Notes for a long time. Unfortunately, the annual subscription has always been too expensive for me and I have only used the free version of Standard Notes. Besides the hosted version of Standard Notes, there is also the possibility to install your own self-hosted Standard Notes instance on your server. With all premium features and even for free. 

Unfortunately, I haven't found any instructions on how to set up Standard Notes together with Docker and Traefik as a reverse proxy. Therefore I decided to explain my setup to you.

## Selfhosted Standard Notes with Docker and Traefik
### Preparation

{{< notice info >}}
If you don't have docker installed yet, you can find instructions for [Ubuntu](/install-docker-on-ubuntu) or [Debian](/docker-install-debian-10). This Guide uses docker-compose to run Traefik, therefore its necessary to also install docker-compose. The two linked guides will help you to setup docker-compose on your own host.

Please also make sure that you already have Traefik installed on your server. If you haven't done that yet, you can find a [step by step guide here](https://ae3.ch/install-use-traefik-reverse-proxy-docker). 
{{< /notice >}}

Frist we create a few files and folders to work with. I like my docker setup to be clean and easy, therefore I create a folder for each docker-compose stack I'm running on my host.

```bash
mkdir -p /opt/containers/standardnotes
mkdir -p /opt/containers/standardnotes/docker
```

Now we download the docker-compose sample config provided by Standard Notes.

```bash
wget -O /opt/containers/standardnotes/docker-compose.yml https://raw.githubusercontent.com/standardnotes/standalone/main/docker-compose.yml
wget -O /opt/containers/standardnotes/.env https://raw.githubusercontent.com/standardnotes/standalone/main/.env.sample
wget -O /opt/containers/standardnotes/docker/auth.env https://raw.githubusercontent.com/standardnotes/standalone/main/docker/auth.env.sample
wget -O /opt/containers/standardnotes/docker/api-gateway.env https://raw.githubusercontent.com/standardnotes/standalone/main/docker/api-gateway.env.sample
```

### Docker-Compose File
Next up we are changing some lines inside the docker-compose sample file. 
```bash
vim /opt/containers/standardnotes/docker-compose.yml
```

To make sure we are always using the latest version of Standard Notes, we need to change the following lines:

```bash
image: standardnotes/syncing-server-js:latest
image: standardnotes/syncing-server-js:latest
image: standardnotes/api-gateway:latest
image: standardnotes/auth:latest
image: redis:latest
```
In the sample config, you will find the exact version instead of ":latest" in each line that needs to be replaced with the above lines.

### Traefik Configuration for Standard Notes
```bash
vim /opt/containers/standardnotes/docker-compose.yml
```

And find the following service: `api-gateway`
You then can delete this service block from your docker-compose.yml file and replace it with the following block.

```bash
  api-gateway:
    image: standardnotes/api-gateway:latest
    depends_on:
      - auth
      - syncing-server-js
    env_file: docker/api-gateway.env
    environment:
      PORT: 3000
      AUTH_JWT_SECRET: '${AUTH_JWT_SECRET}'
    entrypoint: [
      "./wait-for.sh", "auth", "3000",
      "./wait-for.sh", "syncing-server-js", "3000",
      "./docker/entrypoint.sh", "start-web"
    ]
    networks:
      - standardnotes_standalone
      - proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api-gateway.entrypoints=http"
      - "traefik.http.routers.api-gateway.rule=Host(`notes.yourdomain.tld`)"
      - "traefik.http.middlewares.api-gateway-https-redirect.redirectscheme.scheme=https"
      - "traefik.http.routers.api-gateway.middlewares=api-gateway-https-redirect"
      - "traefik.http.routers.api-gateway-secure.entrypoints=https"
      - "traefik.http.routers.api-gateway-secure.rule=Host(`notes.yourdomain.tld`)"
      - "traefik.http.routers.api-gateway-secure.tls=true"
      - "traefik.http.routers.api-gateway-secure.tls.certresolver=http"
      - "traefik.http.routers.api-gateway-secure.service=api-gateway"
      - "traefik.http.services.api-gateway.loadbalancer.server.port=3000"
      - "traefik.docker.network=proxy"
```

It's important to replace `notes.yourdomain.tld` inside the `labels` part with your own domain name. Don't forget to create the appropriate DNS records for your domain. 

At the end of our docker-compose.yml File you need to remove the `networks:` part and replace it with these lines:

```bash
networks:
  standardnotes_standalone:
    name: standardnotes_standalone
  proxy:
    external: true
```

The network must be identical to the network you used for Traefik. In my case it's `proxy`. 
If you followed my instructions for installing Traefik, you can simply copy and paste this part. 

Your `docker-compose.yml` file should now look like this:

```bash
version: '3.8'
services:
  syncing-server-js:
    image: standardnotes/syncing-server-js:latest
    depends_on:
      - db
      - cache
    entrypoint: [
      "./wait-for.sh", "db", "3306",
      "./wait-for.sh", "cache", "6379",
      "./docker/entrypoint.sh", "start-web"
    ]
    env_file: .env
    environment:
      PORT: 3000
    restart: unless-stopped
    networks:
      - standardnotes_standalone

  syncing-server-js-worker:
    image: standardnotes/syncing-server-js:latest
    depends_on:
      - db
      - cache
      - syncing-server-js
    entrypoint: [
      "./wait-for.sh", "db", "3306",
      "./wait-for.sh", "cache", "6379",
      "./wait-for.sh", "syncing-server-js", "3000",
       "./docker/entrypoint.sh", "start-worker"
    ]
    env_file: .env
    environment:
      PORT: 3000
    restart: unless-stopped
    networks:
      - standardnotes_standalone

  api-gateway:
    image: standardnotes/api-gateway:latest
    depends_on:
      - auth
      - syncing-server-js
    env_file: docker/api-gateway.env
    environment:
      PORT: 3000
      AUTH_JWT_SECRET: '${AUTH_JWT_SECRET}'
    entrypoint: [
      "./wait-for.sh", "auth", "3000",
      "./wait-for.sh", "syncing-server-js", "3000",
      "./docker/entrypoint.sh", "start-web"
    ]
    networks:
      - standardnotes_standalone
      - proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api-gateway.entrypoints=http"
      - "traefik.http.routers.api-gateway.rule=Host(`notes.yourdomain.tld`)"
      - "traefik.http.middlewares.api-gateway-https-redirect.redirectscheme.scheme=https"
      - "traefik.http.routers.api-gateway.middlewares=api-gateway-https-redirect"
      - "traefik.http.routers.api-gateway-secure.entrypoints=https"
      - "traefik.http.routers.api-gateway-secure.rule=Host(`notes.yourdomain.tld`)"
      - "traefik.http.routers.api-gateway-secure.tls=true"
      - "traefik.http.routers.api-gateway-secure.tls.certresolver=http"
      - "traefik.http.routers.api-gateway-secure.service=api-gateway"
      - "traefik.http.services.api-gateway.loadbalancer.server.port=3000"
      - "traefik.docker.network=proxy"

  auth:
    image: standardnotes/auth:latest
    depends_on:
      - db
      - cache
      - syncing-server-js
    entrypoint: [
      "./wait-for.sh", "db", "3306",
      "./wait-for.sh", "cache", "6379",
      "./wait-for.sh", "syncing-server-js", "3000",
      "./docker/entrypoint.sh", "start-web"
    ]
    env_file: docker/auth.env
    environment:
      PORT: 3000
      DB_HOST: '${DB_HOST}'
      DB_REPLICA_HOST: '${DB_REPLICA_HOST}'
      DB_PORT: '${DB_PORT}'
      DB_DATABASE: '${DB_DATABASE}'
      DB_USERNAME: '${DB_USERNAME}'
      DB_PASSWORD: '${DB_PASSWORD}'
      DB_DEBUG_LEVEL: '${DB_DEBUG_LEVEL}'
      DB_MIGRATIONS_PATH: '${DB_MIGRATIONS_PATH}'
      REDIS_URL: '${REDIS_URL}'
      AUTH_JWT_SECRET: '${AUTH_JWT_SECRET}'
    networks:
      - standardnotes_standalone

  auth-worker:
    image: standardnotes/auth:latest
    depends_on:
      - db
      - cache
      - auth
    entrypoint: [
      "./wait-for.sh", "db", "3306",
      "./wait-for.sh", "cache", "6379",
      "./wait-for.sh", "auth", "3000",
      "./docker/entrypoint.sh", "start-worker"
    ]
    env_file: docker/auth.env
    environment:
      PORT: 3000
      DB_HOST: '${DB_HOST}'
      DB_REPLICA_HOST: '${DB_REPLICA_HOST}'
      DB_PORT: '${DB_PORT}'
      DB_DATABASE: '${DB_DATABASE}'
      DB_USERNAME: '${DB_USERNAME}'
      DB_PASSWORD: '${DB_PASSWORD}'
      DB_DEBUG_LEVEL: '${DB_DEBUG_LEVEL}'
      DB_MIGRATIONS_PATH: '${DB_MIGRATIONS_PATH}'
      REDIS_URL: '${REDIS_URL}'
      AUTH_JWT_SECRET: '${AUTH_JWT_SECRET}'
    networks:
      - standardnotes_standalone

  db:
    image: mysql:5.6
    environment:
      MYSQL_DATABASE: '${DB_DATABASE}'
      MYSQL_USER: '${DB_USERNAME}'
      MYSQL_PASSWORD: '${DB_PASSWORD}'
      MYSQL_ROOT_PASSWORD: '${DB_PASSWORD}'
    ports:
      - 3306
    restart: unless-stopped
    command: --default-authentication-plugin=mysql_native_password --character-set-server=utf8 --collation-server=utf8_general_ci
    volumes:
      - ./data/mysql:/var/lib/mysql
      - ./data/import:/docker-entrypoint-initdb.d
    networks:
      - standardnotes_standalone

  cache:
    image: redis:latest
    volumes:
      - ./data/redis/:/data
    ports:
      - 6379
    networks:
      - standardnotes_standalone

networks:
  standardnotes_standalone:
    name: standardnotes_standalone
  proxy:
    external: true
```

### Customize the Environment Files

You may have noticed that in the docker-compose file, many values are specified with a variable. 
These variables are filled by an environment file.

Now we edit the sample environment files and set our own values. 

```bash
vim /opt/containers/standardnotes/.env
```

We need to change the following values.

* AUTH_JWT_SECRET
* DB_PASSWORD

For the first one you need to generate a random string:

```bash
openssl rand -hex 32
6d1e6f3dffe6629873e08ab33793fbf80cefbf7c9f8a7de848fa03afc0b3310f
```

For the `DB_PASSWORD` you can use a random and strong Passwort. 

At the end, these two lines should like this:

```bash
AUTH_JWT_SECRET=6d1e6f3dffe6629873e08ab33793fbf80cefbf7c9f8a7de848fa03afc0b3310f

[...]

DB_PASSWORD=1cd0a6041f4656ff9455fe
```

Save your changes and go on with the next .env File. 
This time we are editing the sample `auth.env`. 

```bash
vim /opt/containers/standardnotes/docker/auth.env.
```

The following lines need to be changed:

* JWT_SECRET
* LEGACY_JWT_SECRET
* PSEUDO_KEY_PARAMS_KEY
* ENCRYPTION_SERVER_KEY

Please use the openssl command to generate a random string for all four lines.
```bash
openssl rand -hex 32
```

Save your changes. Done. 

## Start the Standard Notes docker-compose stack
```bash
docker-compose -f /opt/containers/standardnotes/docker-compose.yml up -d
```

You can now visit `https://notes.yourdomain.tld/healthcheck` to make sure your installation is working. 
It should display a `OK` if everything is all right.

## Create a Standard Notes Account
Next up you need to create your account on you'r fresh new self-hosted Standard Notes. 

The easiest way to do this is to use [one of the desktop clients](https://standardnotes.com/) of Standard Notes. I use the Mac client, but you can use whichever client you want. 

In the lower left corner of the app, click on **Account**.
A login window will appear. Here we select **Register**.

![standard-notes-register-1](/images/standard-notes-register-1.png "standard-notes-register-1")

Now it is important that you click on **Advanced options** at the bottom. Here you have to enter your own domain under which your Standard Notes server runs. 
For example https://notes.yourdomain.tld.

![standard-notes-register-2](/images/standard-notes-register-2.png "standard-notes-register-2")

Please make sure that you save your password in a safe place. You will not be able to reset it in the future. 

That's it, from now on you can log in with your account on all devices.

![standard-notes-interface](/images/standard-notes-interface.png "standard-notes-interface")

But remember to always specify your own server under **Advanced Options**, only then the connection to your own instance will be established. 