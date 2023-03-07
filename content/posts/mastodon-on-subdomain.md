---
title: "Hosting Mastodon on a currently used domain name"
date: 2022-11-24 14:04:36+01:00
# aliases: ["/first"]
tags:
    - docker
    - traefik
    - mastodon
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
    image: "https://oggen.ae3.ch/api/ae3?blogtitle=Hosting%20Mastodon%20on%20a%20currently%20used%20domain%20name" # image path/url
    alt: "" # alt text
    caption: "" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---
You already use your domain for a company website and now you want to use Mastodon with your domain?

It is possible to install Mastodon on a subdomain (`mastodon.example.com`) and still be found in [Fediverse](https://en.wikipedia.org/wiki/Fediverse) under `example.com`. 
Mastodon is a decentralized microblogging network, that is similar to Twitter, and can be self-hosted on your own server. 

# Hosting Mastodon on a currently used domain name
## Environment Settings

{{< notice note >}}
By the way, if you haven't installed Mastodon on your server yet, you can follow my instructions to [install Mastodon](/mastodon-docker-traefik) on your server using Docker and Traefik as a reverse proxy. 
{{< /notice >}}

In order to perform the following steps, you must have your Mastodon instance installed on your subdomain.

If you followed my instructions for installing Mastodon with Docker, you will find a file `.env.production` in the directory where Mastodon is started with docker-compose. 
We now open this file and set the following two environment variables:

```
LOCAL_DOMAIN=example.com
WEB_DOMAIN=mastodon.example.com
```

For the first value [LOCAL_DOMAIN](https://docs.joinmastodon.org/admin/config/#local_domain) you use your domain name without the subdomain under which Mastodon really runs. 
This allows you to specify your username with `@username@example.com` instead of `@username@subdomain.example.com`. 

As [WEB_DOMAIN](https://docs.joinmastodon.org/admin/config/#web_domain) you now write down your subdomain where you originally installed Mastodon. 

For these adjustments to work, you now need to restart Mastodon once. 

After setting these two values, you can use Mastodon with a domain that is already running a completely different website without having to move anything. 