---
layout: default 
title: Debug
has_children: false
nav_order: 90
has_toc: false
---

# Debug

## NFtable

```
conntrack -L

conntrack -L | grep OFFLOAD

nft -a list ruleset

nft monitor trace
```

## DHCP

```
tcpdump -i wan port 67 or port 68 -e -n -v
 
tcpdump -i wan port 546 or port 547 -e -n -v
```