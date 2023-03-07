---
title: "Systemd timer as a cron replacement"
date: 2022-12-29 23:35:48+01:00
# aliases: ["/first"]
tags:
    - debian
    - linux
    - systemd
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
    image: "https://oggen.ae3.ch/api/ae3?blogtitle=Systemd%20timer%20as%20a%20cron%20replacement" # image path/url
    alt: "Systemd timer as a cron replacement" # alt text
    caption: "Systemd timer as a cron replacement" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---
Instead of using a cronjob placed in our crontab, we can make use of a systemd timer on a modern linux system which is running systemd. 

For a systemd timer to work, it needs two parts. 
One part consists of a `.timer` file, and a `.service` file. 

The `.timer` file specifies when our service should be started. Just like a cronjob we can specify here on which days, in which week or at which time our service should be started. 
Our `.service` file contains all the details about the script or command we want to run. 

## Example systemd timer file

`/etc/systemd/system/example.timer`

```
[Unit]
Description=restic backup timer

[Timer]
OnCalendar=*-*-* */12:00:00

[Install]
WantedBy=basic.target
```

The syntax of systemd timers is a little bit different from a cronjob. In this example we start our service with `OnCalendar` every 12h. 

There are many different timer settings that execute the service at different times. 

For example `OnBootSec=1h 30m` would start our service 1h 30m after the last boot of the system. One time only. So rather not suitable for a cronjob. 

You can find a very simple explanation and many more examples on the following page at [silentlad.com](https://silentlad.com/systemd-timers-oncalendar-(cron)-format-explained) or on the [systemd.timer manpage](https://manpages.debian.org/testing/systemd/systemd.timer.5.en.html).

## Example systemd service file

`/etc/systemd/system/example.service`

```
[Unit]
Description=restic backup cron

[Service]
ExecStart=/usr/bin/python3 /opt/restic/resticBackup.py backup
```

`ExecStart` can be used to run a command or a script. 
In this example this systemd service would run a python script to create a restic backup of our system.

It is important in this example that the timer and the service both have the same name, in this case: `example.timer` and `example.service`. 

## Enable systemd timer

As soon as you have created your `.timer` and `.service` file you need to enable them once. 

`systemctl enable --now example.timer`

`--now` starts the timer immediately after entering the command. 
Otherwise we would have to wait for the next reboot before the timer is started. 

For testing, it is also possible to restart the timer with `systemctl restart example.timer`. The service is then executed immediately and then repeated after n time as specified in the timer.

With the following command you can see when your service will be executed next time and when the service was started last time. 

`systemctl list-timers`

![systemd_timer_list](/images/systemd_timer_list.png "systemd_timer_list")