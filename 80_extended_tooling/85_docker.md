---
layout: default 
title: Docker
parent: Extended tooling
has_children: false
nav_order: 85
has_toc: false
---

# Docker

## Install

```bash
apt update

apt install ca-certificates curl

install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null

apt update

apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

{: .info }
> Optional: Add access to docker to a particular user.

```bash
groupadd docker

usermod -aG docker <USER>
```

{: .important }
> Replace `<USER>` with the proper value.

## Remove docker from updating iptable / nftable

`nano /etc/docker/daemon.json`

```json
{
        "iptables": false
}
```

`systemctl restart docker.service`