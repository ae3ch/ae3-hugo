---
title: "How to install docker-compose?"
date: 2021-04-28 11:25:26+01:00
# aliases: ["/first"]
tags:
    - docker
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
    image: "https://oggen.ae3.ch/api/ae3?blogtitle=How%20to%20install%20docker-compose?" # image path/url
    alt: "How to install docker-compose?" # alt text
    caption: "How to install docker-compose?" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---
Managing multiple docker containers on CLI could be a painful task, docker-compose is an easy to use solution to manage all your docker containers inside a docker-compose.yml file. You can run multiple docker containers inside one compose file. In this Guide I will show you how to install docker-compose on your Debian or Ubuntu server.

## Install docker-compose on Linux
Make sure you have `curl` installed on your system. For Debian or Ubuntu you can do so, by running: `apt install curl`.

Now you are ready to download the current version of docker-compose directly from their [github repo](https://github.com/docker/compose/releases).
```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

To allow docker-compose to run, we need to set the excutable flag to `/usr/local/bin/docker-compose`.
```bash
sudo chmod +x /usr/local/bin/docker-compose
```

That's it! You can check if docker-compose is installed correctly by running the following command. You should then get the current version as an output. 
```bash
$ docker-compose version
docker-compose version 1.28.5, build c4eb3a1f
```

## How to run docker-compose.yml?
To start your docker-compose.yml stack, make sure you are in the same directory as your compose file is. Then simply run:
```bash
docker-compose up -d
```
This command will start your docker-compose stack and move it to background. This means your stack is running as long as you want, even if you are closing your SSH connection. To stop your stack use `docker-compose stop`. 