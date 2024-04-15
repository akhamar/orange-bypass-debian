---
layout: default 
title: Routing
has_children: false
nav_order: 60
has_toc: false
---

# Routing

## Enable nftables

`systemctl enable nftables.service`

## Configure nftables

`nano /etc/nftables.conf`

```bash
#!/usr/sbin/nft -f

flush ruleset

define LAN_IPV4_SUBNET  = { 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 }
define LAN_IPV6_SUBNET  = { xxxx:xxxx:xxxx:xxxx::/64 }

# NAT
table ip nat {

        chain prerouting {
                type nat hook prerouting priority 0; policy accept;

                #############################################
                # Example Forward
                # WAN(80/443) => IP LAN(80/443)
                #iifname "vlan832" tcp dport { http, https } dnat to 192.168.1.xxx comment "Web Redirect"
        }

        chain input {
                type nat hook input priority 0; policy accept;
                #counter comment "count accepted packets"
        }

        chain output {
                type nat hook output priority 0; policy accept;
                #counter comment "count accepted packets"
        }

        chain postrouting {
                type nat hook postrouting priority 0; policy accept;

                oifname "vlan832" masquerade;

                #counter comment "count accepted packets"
                #counter log prefix "nft#nat: "
        }

}


# Filter IPV4
table ip filter {
        chain input {
                type filter hook input priority filter; policy drop;
                iif lo accept
                ct state { related, established } accept
                iifname "lan" ip saddr $LAN_IPV4_SUBNET accept                  # From LAN to WAN
        }
        chain forward {
                type filter hook forward priority filter; policy drop;
                ct state { related, established } accept
                ct state invalid drop
                iifname "lan" ct state new accept                               # From LAN to WAN (NAT)
                oifname "lan" ip daddr $LAN_IPV4_SUBNET accept                  # From WAN to LAN
        }
        chain output {
                type filter hook output priority filter; policy accept;
        }
}


# Filter IPV6
table ip6 filter {
        chain input {
                type filter hook input priority filter; policy drop;
                iif lo accept
                ct state { related, established } accept
                iifname "lan" ip6 saddr fe80::/64 accept                                                                                        # Accept ip6 local
                iifname "lan" ip6 saddr $LAN_IPV6_SUBNET accept                                                                                 # Accept ip6 subnet

                # Orange Advert & Solicit
                iifname "vlan832" ip6 daddr fe80::/64 udp dport { 546 } accept                                                                  # Accept Orange DHCP Advertise & Reply
                iifname "vlan832" icmpv6 type { nd-neighbor-solicit, nd-router-advert, nd-neighbor-advert } accept                              # Solicit, Advert from Orange on WAN
        }
        chain forward {
                type filter hook forward priority filter; policy drop;
                ct state { related, established } accept
                ct state invalid drop
                iifname "lan" ct state new accept

                #############################################
                # Example IPV6 Filtering
                #############################################
                # Accept ping on all IPV6 in subnet
                ip6 daddr $LAN_IPV6_SUBNET icmpv6 type { echo-request } accept
        }
        chain output {
                type filter hook output priority filter; policy accept;
        }
}
```

{: .important }
> Replace placeholders value with proper values

## Configure nftables with fastpath (optional)

This part is totally optional and doesn't need to be done for the connection to work.

It is more an optimisation (especially for XGS-PON) than anything else.

{: .warning }
> Adding fastpath on nftable explicitely declare lan and vlan832
>
> Thus making impossible to load nftable rule that include fastpath unless vlan832 iface is up
>
> To prevent that we will use an update script that add those rules when the vlan832 is up

`nano /etc/network/inject_flowtable_fastpath`

```bash
#!/bin/bash

# Reload nftable
nft -f /etc/nftables.conf

# Add flowtable fastpath
nft add "flowtable ip filter fastpath { hook ingress priority 0; devices = { lan, vlan832 }; }"
nft add "flowtable ip6 filter fastpath { hook ingress priority 0; devices = { lan, vlan832 }; }"
echo "flowtable fastpath added to nftable rules"

# Add flowtable usage
nft insert "rule ip filter forward ct state { related, established } meta l4proto { tcp, udp } flow offload @fastpath;"
nft insert "rule ip6 filter forward ct state { related, established } meta l4proto { tcp, udp } flow offload @fastpath;"
echo "flowtable fastpath usage added to nftable rules"
```

`chmod 750 /etc/network/inject_flowtable_fastpath`

Then edit interface WAN

`nano /etc/network/interfaces.d/wan`

Add `up /etc/network/inject_flowtable_fastpath` to `vlan832` interface

```bash
# VLAN832
auto vlan832
allow-hotplug vlan832
iface vlan832 inet manual

        # Bind vlan
        vlan-raw-device wan

        # LiveBox mac address
        hw-mac-address ac:84:c9:0e:f5:e8

        # Wait for ONU to be UP
        up /etc/network/wait_for_wan

        # Add nftable fastpath rules
        up /etc/network/inject_flowtable_fastpath

        # Generate Orange Options (user-class, vendor-class, option 90)
        up /etc/dhcp/dhclient-orange-generator

...
```

## Configure nftables with trace (optional / debug)

### Inject trace rules

`nano /etc/network/inject_trace_nft`

```bash
#!/bin/bash

# Inject chain
nft add "chain ip nat trace_chain { type filter hook prerouting priority -1; }"
nft add "chain ip filter trace_chain { type filter hook prerouting priority -1; }"
nft add "chain ip6 filter trace_chain { type filter hook prerouting priority -1; }"
echo "chain trace injected"

# Inject trace rules
nft add "rule ip nat trace_chain meta nftrace set 1"
nft add "rule ip filter trace_chain meta nftrace set 1"
nft add "rule ip6 filter trace_chain meta nftrace set 1"
echo "trace injected"
```

`chmod 750 /etc/network/inject_trace_nft`

`/etc/network/inject_trace_nft`

`nft monitor trace`

### Remove trace rules

`nano /etc/network/remove_trace_nft`

```bash
#!/bin/bash

# Remove trace rules
nft delete chain ip nat trace_chain
nft delete chain ip filter trace_chain
nft delete chain ip6 filter trace_chain
echo "chain trace removed"
```

`chmod 750 /etc/network/remove_trace_nft`

`/etc/network/remove_trace_nft`


## IP forward

`nano /etc/sysctl.conf`

```
# IP forwarding (IPV4 / IPV6)
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1

# Desactivate accepting Router Advertisements
net.ipv6.conf.lan.accept_ra=0

# Accept accepting Router Advertisements even when 'net.ipv6.conf.all.forwarding=1' (force Solicitation/Advertisements true)
net.ipv6.conf.default.accept_ra=2
```