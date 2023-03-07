---
title: "macOS M1 fix incorrect auto date and time after dead battery"
date: 2022-07-10 14:26:00+01:00
# aliases: ["/first"]
tags:
    - macos
    - m1
    - apple
    - macbook
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
    image: "https://oggen.ae3.ch/api/ae3?blogtitle=macOS%20M1%20fix%20incorrect%20auto%20date%20and%20time%20after%20dead%20battery" # image path/url
    alt: "macOS M1 fix incorrect auto date and time after dead battery" # alt text
    caption: "macOS M1 fix incorrect auto date and time after dead battery" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---
When you completely discharge a MacBook with an M1 processor and the device runs out of power, it will lose the date and time on the next boot.

Normally there is a backup battery on a mainboard, which keeps the time and date in such a case. I cannot yet say what Apple has done here exactly and why the problem occurs. But I have a solution for you. 

I recently faced the same problem and tried to set the Mac to automatic date and time setting via System Preferences > Date & Time.

Unfortunately, this does not seem to work under macOS Monterey (12.4). I have received a workaround from Apple Support to set the time and date correctly again via the automatic adjustment. 

## How to fix wrong date and time on M1 MacBooks 

1. disable auto date and time in system settings
2. open Finder and "Go to folder" `/var/db`
3. find Folder `timed` right click and "Get Info", set permissions to "Allow read write access to everyone"
4. open `timed` folder and remove the following config file: `com.apple.timed.plist` 
5. reboot
6. re-enable auto date and time in system settings

You should now have a correct and working auto date and time setting for your M1 MacBook Pro.