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

{: .info }
> To make tcmdump whireshark capture or pcap, just add `-w file_name` at the end of each tcpdump command.
>
> Replace `<X_IFACE_NAME_X>` with your interface name. Most likely `wan`. Do not capture on a vlan interface.

### DHCP Request / Offer
DHCP v4
```bash
tcpdump -i <X_IFACE_NAME_X> port 67 or port 68 -e -n -vvvvv
```

DHCP v6
```bash 
tcpdump -i <X_IFACE_NAME_X> port 546 or port 547 -e -n -vvvvv
```

Both
```bash
tcpdump -i <X_IFACE_NAME_X> port 67 or port 68 or port 546 or port 547 -e -n -vvvvv
```

### RA / RS
```bash
tcpdump -vvvvv -ttt -i <X_IFACE_NAME_X> 'icmp6 and (ip6[40] = 134 or ip6[40] = 133)'
```
> 133 : router solicitation
> 
> 134 : router advertisement

### ARP

```bash
tcpdump -vvvvv -ttt -i <X_IFACE_NAME_X> 'arp'
```


### Main ICMP6
```bash
tcpdump -vvvvv -ttt -i <X_IFACE_NAME_X> 'icmp6 and (ip6[40] = 134 or ip6[40] = 133 or ip6[40] = 135 or ip6[40] = 136 or ip6[40] = 129 or ip6[40] = 128 or ip6[40] = 3 or ip6[40] = 2 or ip6[40] = 1)'
```
- unreachable: 1
- too-big: 2
- time-exceeded: 3
- echo-request: 128
- echo-reply: 129
- router-solicitation: 133
- router-advertisement: 134
- neighbor-solicitation: 135
- neighbor-advertisement: 136

### Correct results

#### ARP

![image](https://raw.githubusercontent.com/akhamar/orange-bypass-debian/main/assets/images/arp.png)

#### DHCP V4

![image](https://raw.githubusercontent.com/akhamar/orange-bypass-debian/main/assets/images/dhcp4.png)

#### DHCP V6

![image](https://raw.githubusercontent.com/akhamar/orange-bypass-debian/main/assets/images/dhcp6.png)

#### ICMP6 Router solicitation & Neighbor solicitation

![image](https://raw.githubusercontent.com/akhamar/orange-bypass-debian/main/assets/images/icmp6_ra_rs.png)