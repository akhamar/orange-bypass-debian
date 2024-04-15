---
layout: default 
title: Interfaces rename
parent: Interfaces
has_children: false
nav_order: 31
has_toc: false
---

# Interface rename

## WAN

`nano /etc/systemd/network/10-wan.link`

```bash
[Match]
MACAddress=xx:xx:xx:xx:xx:xx

[Link]
Name=wan
```
where `xx:xx:xx:xx:xx:xx` correspond to the MAC addr of WAN interface


## LAN

`nano /etc/systemd/network/10-lan.link`

```bash
[Match]
MACAddress=xx:xx:xx:xx:xx:xx

[Link]
Name=lan
```
where `xx:xx:xx:xx:xx:xx` correspond to the MAC addr of LAN interface