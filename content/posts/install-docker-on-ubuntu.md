---
title: "Install Docker on Ubuntu 20.04"
date: 2021-04-23 13:49:37+01:00
# aliases: ["/first"]
tags:
    - docker
    - ubuntu
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
    image: "https://oggen.ae3.ch/api/ae3?blogtitle=Install%20Docker%20on%20Ubuntu%2020.04" # image path/url
    alt: "Install Docker on Ubuntu 20.04" # alt text
    caption: "Install Docker on Ubuntu 20.04" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
---
In this How-To we are going to install Docker on Ubuntu 20.04 (Focal). To be able to use Docker to its full extent, we also install [docker-compose](https://docs.docker.com/compose/).
The instructions can be used for Ubuntu 20.04 as well as for all other current Ubuntu installations, including Ubuntu Hirsute 21.04.

There are two ways to install the Docker Engine, one with a regular apt install and one with an install script made by docker.

## Installing Docker with apt
First, we make sure to remove all versions of Docker. This is especially important if you have ever tried to install Docker on your Ubuntu server before.
```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
````
### 1. Preparation
```bash
sudo apt-get update
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```
### 2. Add Docker GPG key
This GPG key verifies the Docker packages that are loaded via apt. This ensures that only the real Docker packages are installed.
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
````

### 3. Add Docker repository.
Now we'll add the Docker stable repository. If you prefer to install Docker from the testing or nightly repo, you can simply replace `stable` with `testing`, or `nightly` respectively. For production use, however, I strongly recommend using the stable version.
```bash
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 4. Install Docker
```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io
```
### 5. Check Docker installation
From now on, Docker is available on your server. Finally, you can use the following Docker image to test if Docker was installed correctly. 
```bash
sudo docker run hello-world
```
If the installation was successful, this container should print a "Hello World" and exit automatically.

## Docker install script
Even easier and faster Docker can be installed automatically on your server with the following command.

{{< notice info >}}
With this type of installation, you must trust that the Docker installation script is really available under the specified link. If there is malicious code in this file, you would execute it unhindered on your server. I recommend you to install Docker with apt.
{{< /notice >}}

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```
## Install docker-compose
Contrary to Docker installation, there is no separate package for docker-compose, which can be installed via the official Docker repo.
The installation is done directly from Github, on the [following page](https://github.com/docker/compose/releases) you can find all docker-compose releases that are available. 

To install the latest version of docker-compose on Ubuntu, the following command is used.

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```
To test the installation of docker-compose, you can use the following command. This should show you the version of docker-compose, if the installation was successful. 
```bash
docker-compose --version
```
