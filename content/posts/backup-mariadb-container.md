---
title: "How to back up MariaDB databases running in a container"
date: 2022-12-23 12:06:22+01:00
# aliases: ["/first"]
tags:
    - docker
    - backup
    - mariadb
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
    image: "https://oggen.ae3.ch/api/ae3?blogtitle=How%20to%20back%20up%20MariaDB%20databases%20running%20in%20a%20container" # image path/url
    alt: "How to back up MariaDB databases running in a container" # alt text
    caption: "How to back up MariaDB databases running in a container" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---
Backups are sometimes not quite present in our everyday life, but if we suddenly face a problem there is nothing but a backup that can save us. 

In the past, I have mounted the data from MariaDB (or MySQL) containers using Docker on a specific Docker volume and backed it up daily.

Depending on the database engine, this data may or may not be consistent. For an installation that has a very large database workload, such backups are almost never consistent and are therefore not an optimal solution. 

We want to make sure that we can restore a complete backup of the database in case of a disaster. 

# How to back up MariaDB databases running in a container

In the following article I will use the MariaDB container from [linuxserver.io](https://linuxserver.io). 

The great thing about linuxserver.io’s containers is that we can extend them with a simple cron mod without having to manually build a container. 

## docker-compose.yml for MariaDB

```
---
version: "2"
services:
  db:
    image: linuxserver/mariadb
    environment:
      - PUID=1000
      - PGID=1000
      - MYSQL_ROOT_PASSWORD=5KZjsGqb3j64it3iFAMv
      - TZ=Europe/London
      - MYSQL_DATABASE=somedatabase
      - MYSQL_USER=someuser
      - MYSQL_PASSWORD=WmaZmg450vrU2Hq7nMYe
      - DOCKER_MODS=linuxserver/mods:universal-cron
    volumes:
      - ./db:/config
    restart: unless-stopped
```

You need to change `MYSQL_ROOT_PASSWORD`, `MYSQL_DATABASE`, `MYSQL_USER` and `MYSQL_PASSWORD` according to your setup. 

The most important part is the activation of the universal-cron Docker mod via the environment variable `DOCKER_MODS`. Now we can set up our own backup cron. 

## MariaDB Backup Cron
In our `docker-compose.yml` we mount the `/config` volume to the `db` directory. 

Now we need to start our container for the first time so that all necessary data is created and the mount point is created. 

`docker compose up -d`

Next we move to the folder `db` and create a new folder here named `backups`. 

Then we open the following file with an editor:
`vim db/crontabs/root`

Here we now add our MariaDB container backup cron. 
In my example the backup is executed daily at 10:30. 
Of course you can adjust this value on your own. You can find help with cronjobs for example at [crontab.guru](https://crontab.guru/) . 

```
30 10 * * * mysqldump -u someuser -pPASSWORD somedatabase > /config/backups/todays-db-backup.sql
```

The file should look like this. 
Please include your username, database name and the password of the database in the command. 
```
# do daily/weekly/monthly maintenance
# min	hour	day	month	weekday	command
*/15	*	*	*	*	run-parts /etc/periodic/15min
0	*	*	*	*	run-parts /etc/periodic/hourly
0	2	*	*	*	run-parts /etc/periodic/daily
0	3	*	*	6	run-parts /etc/periodic/weekly
0	5	1	*	*	run-parts /etc/periodic/monthly
30 10 * * * mysqldump -u someuser -pPASSWORD somedatabase > /config/backups/todays-db-backup.sql
```

Save the file and restart the database container once:

```
docker compose down
docker compose up -d
```

From now on, a MySQL dump of our MariaDB database will be created every day at 10:30am. 

Remember to back up this mysql dump daily with a backup tool of your choice, as it will be automatically overwritten with the latest backup every day at 10:30. 