---
title: "Securing Grav Admin"
date: 2021-04-23 14:15:42+01:00
# aliases: ["/first"]
tags:
    - grav
    - security
    - htaccess
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
    image: "https://oggen.ae3.ch/api/ae3?blogtitle=Securing%20Grav%20Admin" # image path/url
    alt: "Securing Grav Admin" # alt text
    caption: "How to prevent unauthorized access to your GRAV cms" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---
Grav CMS can be used with an additional admin plugin to be able to administer the website without access to the actual file system.

This comes with potential danger, because whoever gains access to the admin plugin has unrestricted access to your Grav installation.

## .htaccess Rule
With the following .htaccess rule you can protect the admin area of Grav with a password.

Simply add the following content to your .htaccess file in the root directory of your Grav installation e.g. /home/username/public_html/.htaccess.

```bash .htaccess
AuthType Basic
AuthName "login"
AuthUserFile "/home/ae4/passwd"
SetEnvIf REQUEST_URI "^/(admin)" PROTECTED

Deny from all
Satisfy any
Allow from env=!PROTECTED

Require valid-user
```
## Create htpasswd file for authentication
In order to be asked for a password when accessing the admin page, you must first create an htpasswd file.
```bash
htpasswd -c /home/ae3/htpasswd username
```
`username` can be replaced by a user of your choice. This command will prompt to enter a random password.

The next time you visit the admin page at domain.tld/admin you will now be prompted to enter a user and password before the regular Grav login appears. Now we have an additional protection, which protects the complete admin area from unauthorized access.