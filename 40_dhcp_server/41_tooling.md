---
layout: default 
title: DHCP Server - Tooling (Optional)
parent: DHCP Server
has_children: false
nav_order: 41
has_toc: false
---

# DHCP Server - Tooling (Optional)

Add manufacturer database for MAC address

`wget http://standards-oui.ieee.org/oui.txt`

`mv oui.txt /usr/local/etc/`

To print out the current ipv4 used lease

`dhcp-lease-list`