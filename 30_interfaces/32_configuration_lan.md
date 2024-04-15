---
layout: default 
title: Interfaces configuration (LAN)
parent: Interfaces
has_children: false
nav_order: 32
has_toc: false
---

# Interfaces configuration (LAN)

`nano /etc/network/interfaces.d/lan`

```bash
# LAN
allow-hotplug lan
iface lan inet static
        address 192.168.1.1/24
```