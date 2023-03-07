---
title: "Docker error response from daemon: AppArmor"
date: 2023-02-07 20:39:46+01:00
# aliases: ["/first"]
tags:
    - docker
    - debian
    - linux
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
    image: "https://oggen.ae3.ch/api/ae3?blogtitle=Docker%20error%20response%20from%20daemon:%20AppArmor" # image path/url
    alt: "Docker error response from daemon: AppArmor" # alt text
    caption: "Docker error response from daemon: AppArmor" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---
Debian users running the latest Docker version 23 currently receive the following error when starting a container:

`Error response from daemon: AppArmor enabled on system but the docker-default profile could not be loaded`

## How to fix AppArmor Docker error

You may also get the full error message:

```
Error response from daemon: AppArmor enabled on system but the docker-default profile could not be loaded: running `apparmor_parser apparmor_parser --version` failed with output: 
error: exec: "apparmor_parser": executable file not found in $PATH
Error: failed to start containers: container-name
```

To fix this AppArmor Error on Debian 11 all you need to do is installing the AppArmor package. 

`apt install apparmor-utils` 

After the package `apparmor-utils` has been installed, it is sufficient to restart the desired container again. 
The error message then disappears and Docker can start the containers again without problems.

Docker has already reacted and put this issue in the known issues section of the [release notes](https://docs.docker.com/engine/release-notes/23.0/#known-issues) of version 23.0. 

## What is AppArmor used for anyway?
If you are wondering what this AppArmor is, I can quote you a short excerpt from the [man page](https://manpages.debian.org/testing/apparmor/apparmor_parser.8.en.html) of `apparmor_parser`:

> apparmor_parser is used as a general tool to compile, and manage AppArmor policy, including loading new apparmor.d(5) profiles into the Linux kernel.
> 
> AppArmor profiles restrict the operations available to processes