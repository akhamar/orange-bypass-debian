---
layout: default 
title: Debug
has_children: false
nav_order: 90
has_toc: false
---

# Debug

## NFtable

```bash
conntrack -L

conntrack -L | grep OFFLOAD

nft -a list ruleset

nft monitor trace
```

## DHCP

### DHCP Request / Offer
```bash
tcpdump -i <X_IFACE_NAME_X> port 67 or port 68 -e -n -v
 
tcpdump -i <X_IFACE_NAME_X> port 546 or port 547 -e -n -v
```

### RA / RS
```bash
tcpdump -vvvv -ttt -i <X_IFACE_NAME_X> 'icmp6 and (ip6[40] = 134 or ip6[40] = 133)'
```
> 133 : router solicitation
> 
> 134 : router advertisement
