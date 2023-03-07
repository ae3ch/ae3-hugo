---
title: "Install and use Traefik as a reverse proxy for Docker"
date: 2021-04-27 16:38:51+01:00
# aliases: ["/first"]
tags:
    - docker
    - traefik
    - proxy
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
    image: "https://oggen.ae3.ch/api/ae3?blogtitle=Install%20and%20use%20Traefik%20as%20a%20reverse%20proxy%20for%20Docker" # image path/url
    alt: "Install and use Traefik as a reverse proxy for Docker" # alt text
    caption: "Install and use Traefik as a reverse proxy for Docker" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---
In this Guide we will install [Traefik](https://doc.traefik.io/traefik/) v2 as a reverse proxy with free LetsEncrypt SSL certificates for our Docker container. With Traefik you will be able to run multiple containers with different domains. This guide offers a ready to use `docker-compose.yml` for a quick start with Traefik.

## Traefik v2 Install
### Preparation

{{< notice info >}}
If you don't have docker installed yet, you can find instructions for [Ubuntu](/install-docker-on-ubuntu) or [Debian](/docker-install-debian-10). This Guide uses docker-compose to run Traefik, therefore its necessary to also install docker-compose. The two linked guides will help you to setup docker-compose on your own host. 
{{< /notice >}}

Frist we create a few files and folders to work with. I like my docker setup to be clean and easy, therefore I create a folder for each docker-compose Stack I'm running on my host.

```bash
mkdir -p /opt/containers/traefik
mkdir /opt/containers/traefik/data
touch /opt/containers/traefik/data/traefik.yml
touch /opt/containers/traefik/data/acme.json
chmod 600 /opt/containers/traefik/data/acme.json
```

To create password protected sites, like our Traefik Dasboard, you will need to install apache2-utils which provides the handy `htpasswd` tool we are using in this Guide. 
```bash
apt-get install apache2-utils
```
### traefik.yml
Next we open our newly created traefik config file with an editor of your choice. 
```bash
vim /opt/containers/traefik/data/traefik.yml
```

```bash
api:
  dashboard: true
entryPoints:
  http:
    address: ":80"
  https:
    address: ":443"
providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
certificatesResolvers:
  http:
    acme:
      email: mail@domain.tld   # CHANGE HERE
      storage: acme.json
      httpChallenge:
        entryPoint: http
```
You need to replace the example email with your own email address. This email address is used to request our free LetsEncrypt SSL certificates. 
In this Guide we are enabling the Traefik Dashboard. This Dashboard is helpful to check if a service is running correct or not. If you are not interested to run Trafik with the Dashboard enabled, you can set `dashboard: false`. I would suggest to leave the Dashboard running, if you are new to Traefik. 

### docker-compose.yml
```bash
version: '3'
services:
  traefik:
    image: traefik:latest
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    networks:
      - proxy
    ports:
      - 80:80
      - 443:443
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./data/traefik.yml:/traefik.yml:ro
      - ./data/acme.json:/acme.json
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.entrypoints=http"
      - "traefik.http.routers.traefik.rule=Host(`traefik.domain.tld`)"
      - "traefik.http.middlewares.traefik-auth.basicauth.users=USER:PASSWORD"
      - "traefik.http.middlewares.traefik-https-redirect.redirectscheme.scheme=https"
      - "traefik.http.routers.traefik.middlewares=traefik-https-redirect"
      - "traefik.http.routers.traefik-secure.entrypoints=https"
      - "traefik.http.routers.traefik-secure.rule=Host(`traefik.domain.tld`)"
      - "traefik.http.routers.traefik-secure.middlewares=traefik-auth"
      - "traefik.http.routers.traefik-secure.tls=true"
      - "traefik.http.routers.traefik-secure.tls.certresolver=http"
      - "traefik.http.routers.traefik-secure.service=api@internal"
networks:
  proxy:
    external: true
```

With our docker-compose.yml we are defining the Traefik docker container with all the settings and config files. To get Traefik up and running you only need to adjust some settings:

* replace both `traefik.domain.tld` with your own domain name. This domain should be a subdomain like traefik.ae3.ch for example. Later you will be able access Traefik Dasboard with this (sub)domain. 
* replace `USER:PASSWORD` with your own credentials. 

You will not be able to write your password as it is, you first need to generate a hash of your password, like so:
```bash
echo $(htpasswd -nb user password) | sed -e s/\\$/\\$\\$/g
```
Replace `user` and `password` with your own credentials you would like to use for Traefik Dashboard. 

Output should look like this:
```bash
user:$$apr1$$zUb/YuK2$$57psQ0U71DlfdHPr0yoHe/
```
Now you can copy the whole line of your own output and replace the `USER:PASSWORD` with your own credentials:

```bash
# before
      - "traefik.http.middlewares.traefik-auth.basicauth.users=USER:PASSWORD"
# after
      - "traefik.http.middlewares.traefik-auth.basicauth.users=user:$$apr1$$zUb/YuK2$$57psQ0U71DlfdHPr0yoHe/"
```
### Create Docker Network for Traefik
It's a good idea to setup a separate docker network that is used by Traefik and all other docker containers you would like to make available by Traefik. 

To create this docker network, all you need to do is paste the following command into your CLI:
```bash
docker network create proxy
```
### Run Traefik
Now we are ready to run our traefik container. Make sure you are located inside the `/opt/containers/traefik` directory and then simply run:
```bash
docker-compose up -d
```
After a few seconds you can check and access your Traefik Dashboard at your custom Domain you entered in your docker-compose.yml (example: traefik.ae3.ch). When you try to visit the Dasboard, you should be prompted by a login form, there you enter your username and password you defined earlier. 

Traefik will automatically generate SSL certificates for all domains you are connecting to Traefik. All certificates are also renewed automatically, this means you don't need to do any manual steps to secure your website with SSL, isn't it fancy huh? :) 

## Use your container with Traefik
To access your own containers with your domain, you need to add some config labels to your existing docker-compose.yml file of your other containers. 
This is an example config, you could use with any docker container you are running.

```bash
   service-name:
     labels:
      - "traefik.enable=true"
      ## replace `service-name` with the name of your docker container, you will need to replace this entry in all lines
      - "traefik.http.routers.service-name.entrypoints=http"
      ## replace `example.domain.tld` with your own domain name, for example `wordpress.domain.ch`
      - "traefik.http.routers.service-name.rule=Host(`example.domain.tld`)"
      - "traefik.http.middlewares.service-name-https-redirect.redirectscheme.scheme=https"
      - "traefik.http.routers.service-name.middlewares=service-name-https-redirect"
      - "traefik.http.routers.service-name-secure.entrypoints=https"
      ## replace `example.domain.tld` with your own domain name, for example `wordpress.domain.ch`
      - "traefik.http.routers.service-name-secure.rule=Host(`example.domain.tld`)"
      - "traefik.http.routers.service-name-secure.tls=true"
      - "traefik.http.routers.service-name-secure.tls.certresolver=http"
      ## replace `service-name` with the name of your docker container
      - "traefik.http.routers.service-name-secure.service=service-name"
      ## if your container is using a different port then `80`, just replace this port with your custom one, for example: `8888`
      - "traefik.http.services.service-name.loadbalancer.server.port=80"
      - "traefik.docker.network=proxy"
```
Make sure your container has the same docker network assignes as traefik has. You can do this by adding the following to your docker-compose.yml
```bash
   service-name:
     networks:
       - proxy
networks:
  proxy:
    external: true
```
### WordPress docker-compose.yml for Traefik
A complete `docker-compose.yml` for WordPress could look like this:

```bash
version: '3.1'

services:

  wordpress:
    image: wordpress
    restart: always
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: exampleuser
      WORDPRESS_DB_PASSWORD: examplepass
      WORDPRESS_DB_NAME: exampledb
    volumes:
      - wordpress:/var/www/html
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.wordpress.entrypoints=http"
      - "traefik.http.routers.wordpress.rule=Host(`wordpress.domain.ch`)"
      - "traefik.http.middlewares.wordpress-https-redirect.redirectscheme.scheme=https"
      - "traefik.http.routers.wordpress.middlewares=wordpress-https-redirect"
      - "traefik.http.routers.wordpress-secure.entrypoints=https"
      - "traefik.http.routers.wordpress-secure.rule=Host(`wordpress.domain.ch`)"
      - "traefik.http.routers.wordpress-secure.tls=true"
      - "traefik.http.routers.wordpress-secure.tls.certresolver=http"
      - "traefik.http.routers.wordpress-secure.service=wordpress"
      - "traefik.http.services.wordpress.loadbalancer.server.port=80"
      - "traefik.docker.network=proxy"
    networks:
      - proxy

  db:
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_DATABASE: exampledb
      MYSQL_USER: exampleuser
      MYSQL_PASSWORD: examplepass
      MYSQL_RANDOM_ROOT_PASSWORD: '1'
    volumes:
      - db:/var/lib/mysql

volumes:
  wordpress:
  db:
 
networks:
  proxy:
    external: true
```