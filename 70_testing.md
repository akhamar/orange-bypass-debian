---
layout: default 
title: Testing
has_children: false
nav_order: 70
has_toc: false
---

# Testing

`ifdown vlan832 && ifup vlan832`

`systemctl restart isc-dhcp-server.service radvd.service nftables.service`

`systemctl status isc-dhcp-server.service radvd.service nftables.service`