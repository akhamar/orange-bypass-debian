---
layout: default 
title: DDClient
parent: Extended tooling
has_children: false
nav_order: 81
has_toc: false
---

# Dynamic DNS with DDClient (optional)

## Install

`apt install ddclient dh-autoreconf`

{: .important }
> Clone the lastest version of ddclient if the current version is not at least 3.11.3_0
>
> You don't need to clone if ddclient is more up to date

{: .info }
> Only needed if ddclient is not at least 3.11.3_0

```
cd /tmp && git clone https://github.com/ddclient/ddclient.git && cd /tmp/ddclient
./autogen
./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var
make
make install
```

## Configuration

`nano /etc/ddclient.conf`

{: .info }
> Example with gandi

```
ssl=yes
force=yes

protocol=gandi
zone=bbbb.cc
password=<PASSWORD>
usev4=ifv4
usev6=ifv6
ifv4=vlan832
ifv6=vlan832
aaaa.bbbb.cc
```

{: .important }
> Replace placeholders value with proper values