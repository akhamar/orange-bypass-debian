---
layout: default 
title: Interfaces configuration (WAN - XGS-PON)
parent: Interfaces
has_children: false
nav_order: 34
has_toc: false
---

# Interfaces configuration (WAN - XGS-PON)

## Interface

### Interface configuration

For XGS-PON it's best to set the speed of the ONU stick manually.

`nano /etc/network/interfaces.d/wan`

```bash
# WAN
auto wan
allow-hotplug wan
iface wan inet static
        address xxx.xxx.xxx.xxx/xx
        link-speed 10000
```

{: .important }
> Replace `xxx.xxx.xxx.xxx/xx` with the ONU stick static IP.