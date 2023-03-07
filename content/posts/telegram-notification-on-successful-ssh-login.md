---
title: "Telegram Notification on successful SSH Login"
date: 2022-12-12 17:46:28+01:00
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
    image: "https://oggen.ae3.ch/api/ae3?blogtitle=Telegram%20Notification%20on%20successful%20SSH%20Login" # image path/url
    alt: "Telegram Notification on successful SSH Login" # alt text
    caption: "Telegram Notification on successful SSH Login" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---
With the following script we monitor the SSH login on a server and send a notification via Telegram API. 

This way you will get a notification in real time when someone logs in to your server via SSH. For me this is a valuable security feature. 

# Telegram Notification on successful SSH Login

## telegram-send Python Script
First, we create the following Python script, which is used to send messages to Telegram's API. 

You can use this script for the SSH-watcher, but also for cronjobs or other commands that send output to the CLI. 

`vim telegram-send.py`

```python
import sys
import requests

TELEGRAM_API_KEY = "<your_api_key>"
TELEGRAM_CHAT_ID = "<your_chat_id>"

message = " ".join(sys.argv[1:])

if not message:
    print("Please provide a message to send.")
    sys.exit(1)

# Use the Telegram API to send the message with markdown formatting
response = requests.get(
    f"https://api.telegram.org/bot{TELEGRAM_API_KEY}/sendMessage?chat_id={TELEGRAM_CHAT_ID}&parse_mode=markdown&text={message}"
)

if response.status_code == 200:
    print("Message sent successfully.")
else:
    print("Failed to send message. Please check your API key and chat ID.")
    sys.exit(1)
```

To use this script, you will need to replace `<your_api_key>` and `<your_chat_id>` with your own API key and chat ID. You can get your API key by creating a new bot with the [BotFather](https://core.telegram.org/bots#6-botfather) and chat ID by starting a conversation with your bot and sending a message to it.

You can then run the script like this:
`python telegram-send.py Message`

Now move the script to a persistent location, such as: 
`mv telegram-send.py /usr/bin/telegram-send.py` 

## /etc/login.d/ Script
Now it is about creating a script which will be executed on every SSH login. 

For this it is enough to put a bash script in the `/etc/login.d/` directory. 
Once a user successfully logs in, the script is called and we are notified of a login via Telegram. 

`vim /etc/profile.d/login-notify.sh`

```bash
#!/bin/bash

login_ip="$(echo $SSH_CONNECTION | cut -d " " -f 1)"
login_date="$(date +"%a %e %b %Y, %R")"
login_name="$(whoami)"

message="*Host:* $HOSTNAME"$'\n'"*User:* $login_name"$'\n'"*IP:* $login_ip"$'\n'"$login_date"

python3 /usr/bin/telegram-send.py "$message"
```

## Exclude yourself from the notification
If you want to exclude your own IP address from the warning, you can use this example. This will not trigger a notification if the SSH login is coming from your own IP address. 

```bash
#!/bin/bash

login_ip="$(echo $SSH_CONNECTION | cut -d " " -f 1)"
login_date="$(date +"%a %e %b %Y, %R")"
login_name="$(whoami)"

message="*Host:* $HOSTNAME"$'\n'"*User:* $login_name"$'\n'"*IP:* $login_ip"$'\n'"$login_date"

if [[ $login_ip != "<your-ip-address>" ]]
then
    python3 /usr/bin/telegram-send.py "$message"
fi
```

Replace `<your-ip-address>` with your own static IP-address. 

## Testing
Open a new Terminal and try to connect with `ssh user@yourserver.tld`. 

![ssh-watcher-telegram-bot](/images/ssh-watcher-telegram-bot.png "ssh-watcher-telegram-bot")

You should receive an instant notification from your Telegram Bot. 