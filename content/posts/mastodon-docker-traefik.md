---
title: "Mastodon with Docker and Traefik"
date: 2023-03-04T10:30:03+00:00
# weight: 1
# aliases: ["/first"]
tags:
    - docker
    - traefik
    - mastodon
author: "ae3"
# author: ["Me", "You"] # multiple authors
draft: false
hidemeta: false
comments: false
description: "Mastodon is a decentralized microblogging network, that is similar to Twitter, and can be self-hosted on your own server."
#canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "https://oggen.ae3.ch/api/ae3?blogtitle=Mastodon%20with%20Docker%20and%20Traefik" # image path/url
    alt: "Mastodon is a decentralized microblogging network, that is similar to Twitter, and can be self-hosted on your own server." # alt text
    caption: "Mastodon is a decentralized microblogging network, that is similar to Twitter, and can be self-hosted on your own server." # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---
Mastodon is a decentralized microblogging network, that is similar to Twitter, and can be self-hosted on your own server. 

Unlike Twitter, Mastodon consists of many different servers that are connected worldwide. This connection is established through the so-called [Fediverse](https://en.wikipedia.org/wiki/Fediverse). 

It doesn't matter if you create your account on your own self-hosted Mastodon instance or if you create an account with one of the numerous providers. Since all servers are connected, you can interact with your account from anywhere. 

Since the sale of Twitter to Elon Musk there is a huge growth of accounts within Mastodon. If you want to get your own Mastodon server, I'll show you how to start your own Mastodon server with Docker and Traefik.

# Mastodon with Docker and Traffic
## Preparation
{{< notice info >}}
If you don't have docker installed yet, you can find instructions for [Ubuntu](/install-docker-on-ubuntu) or [Debian](/docker-install-debian-10). This Guide uses docker compose to run Traefik, therefore its necessary to install the latest docker version available.
{{< /notice >}}

{{< notice info >}}
Please also make sure that you already have Traefik installed on your server. If you haven't done that yet, you can follow my [step-by-step guide](/install-use-traefik-reverse-proxy-docker).
{{< /notice >}}

This guide is tested with the latest **Mastodon** version **v4.0.2**, but it can also be used with all newer versions. 

At first, we create a new folder to work with:

```
mkdir -p /opt/containers/mastodon
```

## docker-compose.yml
Now we are ready to create our docker-compose for Mastodon.

`vim /opt/containers/mastodon/docker-compose.yml`

```
version: '3'
services:
  db:
    restart: unless-stopped
    image: postgres:14-alpine
    shm_size: 256mb
    networks:
      - default
    healthcheck:
      test: ['CMD', 'pg_isready', '-U', 'postgres']
    volumes:
      - ./postgres14:/var/lib/postgresql/data
    environment:
      - 'POSTGRES_HOST_AUTH_METHOD=trust'

  redis:
    restart: unless-stopped
    image: redis:7-alpine
    networks:
      - default
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
    volumes:
      - ./redis:/data

  web:
    image: tootsuite/mastodon
    restart: unless-stopped
    env_file: .env.production
    command: bash -c "rm -f /mastodon/tmp/pids/server.pid; bundle exec rails s -p 3000"
    networks:
      - proxy
      - default
    healthcheck:
      test: ['CMD-SHELL', 'wget -q --spider --proxy=off localhost:3000/health || exit 1']
    depends_on:
      - db
      - redis
    volumes:
      - ./public/system:/mastodon/public/system
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.mastodon.entrypoints=https"
      - "traefik.http.routers.mastodon.rule=(Host(`mastodon.example.com`))"
      - "traefik.http.routers.mastodon.tls=true"
      - "traefik.http.routers.mastodon.tls.certresolver=http"
      - "traefik.http.routers.mastodon.service=mastodon"
      - "traefik.http.services.mastodon.loadbalancer.server.port=3000"
      - "traefik.docker.network=proxy"

  streaming:
    image: tootsuite/mastodon
    restart: unless-stopped
    env_file: .env.production
    command: node ./streaming
    networks:
      - default
      - proxy
    healthcheck:
      test: ['CMD-SHELL', 'wget -q --spider --proxy=off localhost:4000/api/v1/streaming/health || exit 1']
    depends_on:
      - db
      - redis
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.mastodon-api.entrypoints=https"
      - "traefik.http.routers.mastodon-api.rule=(Host(`mastodon.example.com`) && PathPrefix(`/api/v1/streaming`))"
      - "traefik.http.routers.mastodon-api.tls=true"
      - "traefik.http.routers.mastodon-api.tls.certresolver=http"
      - "traefik.http.routers.mastodon-api.service=mastodon-api"
      - "traefik.http.services.mastodon-api.loadbalancer.server.port=4000"
      - "traefik.docker.network=proxy"

  sidekiq:
    image: tootsuite/mastodon
    restart: unless-stopped
    env_file: .env.production
    command: bundle exec sidekiq
    depends_on:
      - db
      - redis
    networks:
      - default
    volumes:
      - ./public/system:/mastodon/public/system
    healthcheck:
      test: ['CMD-SHELL', "ps aux | grep '[s]idekiq\ 6' || false"]
networks:
  proxy:
    external: true
```

{{< notice note >}}
You need to change `mastodon.example.com` to your own domain name.
{{< /notice >}}

## Create .env.production for Mastodon
We are now ready to generate our `.env.production` file, which contains all settings for our own Mastodon Server. 

Instead of editing the default .env file from Mastodons [GitHub repo](https://github.com/mastodon/mastodon), we will use the following setup command, which is an interactive way of generating the config for us. 

```
cd /opt/containers/mastodon
docker compose run web bundle exec rake mastodon:setup
```

{{< notice note >}}
You need to enter your domain name,  and your SMTP-Server address for sending notifications. All the other questions can be skipped with Enter. 
{{< /notice >}}

```
Your instance is identified by its domain name. Changing it afterward will break things.
Domain name: example.com

Single user mode disables registrations and redirects the landing page to your public profile.
Do you want to enable single user mode? No

Are you using Docker to run Mastodon? Yes

PostgreSQL host: db
PostgreSQL port: 5432
Name of PostgreSQL database: postgres
Name of PostgreSQL user: postgres
Password of PostgreSQL user:
Database configuration works! 🎆

Redis host: redis
Redis port: 6379
Redis password:
Redis configuration works! 🎆

Do you want to store uploaded files on the cloud? No

Do you want to send e-mails from localhost? No
SMTP server: mail.example.com
SMTP port: 587
SMTP username: smtp@example.com
SMTP password:
SMTP authentication: plain
SMTP OpenSSL verify mode: none
E-mail address to send e-mails "from": Mastodon <notifications@example.com>
Send a test e-mail with this configuration right now? No

This configuration will be written to .env.production
Save configuration? Yes
Below is your configuration, save it to an .env.production file outside Docker:

# Generated with mastodon:setup on 2022-11-16 17:14:55 UTC

# Some variables in this file will be interpreted differently whether you are
# using docker-compose or not.

LOCAL_DOMAIN=example.com
SINGLE_USER_MODE=false
SECRET_KEY_BASE=XXXXXXXX
OTP_SECRET=XXXXXXXX
VAPID_PRIVATE_KEY=XXXXXXXX
VAPID_PUBLIC_KEY=XXXXXXXX
DB_HOST=db
DB_PORT=5432
DB_NAME=postgres
DB_USER=postgres
DB_PASS=
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=
SMTP_SERVER=smtp.example.com
SMTP_PORT=587
SMTP_LOGIN=smtp@example.com
SMTP_PASSWORD=XXXXXXXX
SMTP_AUTH_METHOD=plain
SMTP_OPENSSL_VERIFY_MODE=none
SMTP_FROM_ADDRESS=Mastodon <notifications@example.com>

It is also saved within this container so you can proceed with this wizard.

Now that configuration is saved, the database schema must be loaded.
If the database already exists, this will erase its contents.
Prepare the database now? Yes
Running `RAILS_ENV=production rails db:setup` ...


Database 'postgres' already exists
Done!

All done! You can now power on the Mastodon server 🐘

Do you want to create an admin user straight away? Yes
Username: admin
E-mail: admin@example.com
Error connecting to Redis on 127.0.0.1:6379 (Errno::ECONNREFUSED)
Error connecting to Redis on 127.0.0.1:6379 (Errno::ECONNREFUSED)
Switching object-storage-safely from green to red because Redis::CannotConnectError Error connecting to Redis on 127.0.0.1:6379 (Errno::ECONNREFUSED)
Error connecting to Redis on 127.0.0.1:6379 (Errno::ECONNREFUSED)
You can login with the password: XXXXXXXX
You can change your password once you login.
```

Stop all remaining containers with the following command:
`docker compose down`

To use the generated config, we have to save it in the file `.env.production`. The file must be stored in the same location as our `docker-compose.yml`

`vim /opt/containers/mastodon/.env.production`

Here you can copy & paste the config from above, example:

```
# Generated with mastodon:setup on 2022-11-16 17:14:55 UTC

# Some variables in this file will be interpreted differently whether you are
# using docker-compose or not.

LOCAL_DOMAIN=example.com
SINGLE_USER_MODE=false
SECRET_KEY_BASE=XXXXXXXX
OTP_SECRET=XXXXXXXX
VAPID_PRIVATE_KEY=XXXXXXXX
VAPID_PUBLIC_KEY=XXXXXXXX
DB_HOST=db
DB_PORT=5432
DB_NAME=postgres
DB_USER=postgres
DB_PASS=
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=
SMTP_SERVER=smtp.example.com
SMTP_PORT=587
SMTP_LOGIN=smtp@example.com
SMTP_PASSWORD=XXXXXXXX
SMTP_AUTH_METHOD=plain
SMTP_OPENSSL_VERIFY_MODE=none
SMTP_FROM_ADDRESS=Mastodon <notifications@example.com>
```

Save the file and close `vim` with `ESC` and type `:wq`.

## Fix Folder Permissions
In the current Mastodon container build, we need to run an additional step to fix the folder permissions of the `public` folder inside our docker compose stack. 

The error inside mastodon docker logs looks similar to this:
```shell
Errno::EACCES (Permission denied @ dir_s_mkdir - /opt/mastodon/public/system/cache):
```

Let’s fix this for now:
```shell
cd /opt/containers/mastodon
chown -R 991:991 ./public
```

{{< notice info >}}
If you use this guide in the future, this step may no longer be necessary. Currently, Mastodon starts without this customization, but can not connect to other users and can not store files on your server.
{{< /notice >}}

## Start Mastodon Server

`docker compose -f /opt/containers/mastodon/docker-compose.yml up -d` 

{{< notice note >}}
Please not that the startup takes a few minutes, you can try to access your website after 1-5min. 
{{< /notice >}}

## Optional Settings

### Use Mastodon as a single user
If you want to use Mastodon as the only user, I recommend you to create your personal user after starting Mastodon and then deactivate the registration on your instance with your admin account.

You can do so by logging in as your admin user and going to `/admin/settings/edit`. 

![mastodon-disable-registration](/images/mastodon-disable-registration.png "mastodon-disable-registration")