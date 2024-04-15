---
layout: default 
title: Routing - nftable advanced
parent: Routing
has_children: false
nav_order: 61
has_toc: false
---

# NFTABLE

This optional part replace the files created for nftable:
- /etc/nftables.conf
- /etc/network/inject_flowtable_fastpath
- /etc/network/inject_trace_nft
- /etc/network/remove_trace_nft

## Main configuration (with counter and comment)

`nano /etc/nftables.conf`

```bash
#!/usr/sbin/nft -f

flush ruleset

# Subnets
define LAN_IPV4_SUBNET  = { 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 }
define LAN_IPV6_SUBNET  = { xxxx:xxxx:xxxx:xxxx::/64 }
define VPN_IPV4_SUBNET  = { xxx.xxx.xxx.xxx/xx, xxx.xxx.xxx.xxx/xx }

# IFace
define WAN_IFACE        = { "vlan832" }
define LAN_IFACE        = { "lan" }     # { "lan1", "lan2", "lanN" }
define VPN_IFACE        = { "xxxxx_vpn1_ifacename_xxxxx0", "xxxxx_vpn2_ifacename_xxxxx0" }

# Hosts
define IPV4_HOST_1     = { xxx.xxx.xxx.xxx }
define IPV6_HOST_1     = { xxxx:xxxx:xxxx:xxxx::xxxx/128 }

# NAT
table ip nat {

        chain prerouting {
                type nat hook prerouting priority 0; policy accept;

                #############################################
                # Example Forward
                # WAN(80/443) => IP LAN(80/443)
                #iifname $WAN_IFACE tcp dport { http, https } dnat to 192.168.1.xxx comment "Web Redirect"
                #iifname $WAN_IFACE tcp dport { ssh } counter dnat to $IPV4_HOST_1:22 comment "nat.prerouting ssh redirect accepted"
        }

        chain input {
                type nat hook input priority 0; policy accept;
                counter comment "nat.input accepted"
        }

        chain output {
                type nat hook output priority 0; policy accept;
                counter comment "nat.output accepted"
        }

        chain postrouting {
                type nat hook postrouting priority 0; policy accept;
                counter comment "nat.postrouting accepted"

                oifname $WAN_IFACE counter masquerade comment "nat.postrouting masquerade accepted"

                #counter comment "count accepted packets"
                #counter log prefix "nft#nat: "
        }

}


# Filter IPV4
table ip filter4 {
        chain input {
                type filter hook input priority filter; policy drop;
                iif lo accept
                ct state { related, established } counter accept comment "filter4.input related/established accepted"
                iifname $LAN_IFACE ip saddr $LAN_IPV4_SUBNET counter accept comment "filter4.input From LAN Subnet accepted"            # From LAN Subnet on LAN IFace (Accept)
                iifname $VPN_IFACE ip saddr $VPN_IPV4_SUBNET counter accept comment "filter4.input From VPN Subnet accepted"            # From VPN Subnet on VPN IFace (Accept)
                counter comment "filter4.input droped"
        }
        chain forward {
                type filter hook forward priority filter; policy drop;
                ct state { related, established } counter accept comment "filter4.forward related/established accepted"
                ct state invalid counter drop comment "filter4.forward invalid dropped"
                iifname $LAN_IFACE ct state new counter accept comment "filter4.forward LAN to WAN accepted"                            # From LAN to WAN (NAT)
                oifname $LAN_IFACE ip daddr $LAN_IPV4_SUBNET counter accept comment "filter4.forward WAN to LAN accepted"               # From WAN to LAN
                counter comment "filter4.forward droped"
        }
        chain output {
                type filter hook output priority filter; policy accept;
                counter comment "filter4.output accepted"
        }
}


# Filter IPV6
table ip6 filter6 {
        chain input {
                type filter hook input priority filter; policy drop;
                iif lo accept
                ct state { related, established } counter accept comment "filter6.input related/established accepted"
                iifname $LAN_IFACE ip6 saddr fe80::/64 counter accept comment "filter6.input From LAN Subnet accepted"                  # From LAN Subnet on LAN IFace (Accept)
                iifname $LAN_IFACE ip6 saddr $LAN_IPV6_SUBNET counter accept comment "filter6.input From LAN Subnet accepted"           # From LAN Subnet on LAN IFace (Accept)

                # Orange Advert & Solicit
                iifname $WAN_IFACE ip6 daddr fe80::/64 udp dport { 546 } counter accept comment "filter6.input Orange DHCP Adv/Rpl accepted"                                            # Accept Orange DHCP Advertise & Reply
                iifname $WAN_IFACE icmpv6 type { nd-neighbor-solicit, nd-router-advert, nd-neighbor-advert } counter accept comment "filter6.input Orange DHCP Sol/Adv accepted"        # Solicit, Advert from Orange on WAN

                counter comment "filter6.input droped"
        }
        chain forward {
                type filter hook forward priority filter; policy drop;
                ct state { related, established } counter accept comment "filter6.forward related/established accepted"
                ct state invalid counter drop comment "filter6.forward invalid dropped"
                iifname $LAN_IFACE ct state new counter accept comment "filter6.forward LAN to WAN accepted"

                #############################################
                # Example IPV6 Filtering
                #############################################
                # Accept ping on all IPV6 in subnet
                ip6 daddr $LAN_IPV6_SUBNET icmpv6 type { echo-request } counter accept comment "filter6.forward icmp accepted"
                ip6 daddr $IPV6_HOST_1 tcp dport { ssh } counter accept comment "filter6.forward ssh accepted"

                counter comment "filter6.forward droped"
        }
        chain output {
                type filter hook output priority filter; policy accept;
                counter comment "filter6.output accepted"
        }
}
```



## Fastpath injection

`nano /etc/network/inject_flowtable_fastpath`

```bash
#!/bin/bash

# Reload nftable
nft -f /etc/nftables.conf

# Add flowtable fastpath
nft add "flowtable ip filter4 fastpath { hook ingress priority 0; devices = { lan, vlan832 }; }"
nft add "flowtable ip6 filter6 fastpath { hook ingress priority 0; devices = { lan, vlan832 }; }"
echo "flowtable fastpath added to nftable rules"

# Add flowtable usage
nft insert "rule ip filter4 forward ct state { related, established } meta l4proto { tcp, udp } counter flow offload @fastpath comment \"filter4.forward fastpath\""
nft insert "rule ip6 filter6 forward ct state { related, established } meta l4proto { tcp, udp } counter flow offload @fastpath comment \"filter6.forward fastpath\""
echo "flowtable fastpath usage added to nftable rules"
```

`chmod 750 /etc/network/inject_flowtable_fastpath`



## Trace injection

`nano /etc/network/inject_trace_nft`

```bash
#!/bin/bash

# Inject chain
nft add "chain ip nat trace_chain { type filter hook prerouting priority -1; }"
nft add "chain ip filter4 trace_chain { type filter hook prerouting priority -1; }"
nft add "chain ip6 filter6 trace_chain { type filter hook prerouting priority -1; }"
echo "chain trace injected"

# Inject trace rules
nft add "rule ip nat trace_chain meta nftrace set 1"
nft add "rule ip filter4 trace_chain meta nftrace set 1"
nft add "rule ip6 filter6 trace_chain meta nftrace set 1"
echo "trace injected"
```

`chmod 750 /etc/network/inject_trace_nft`



## Trace suppression

`nano /etc/network/remove_trace_nft`

```bash
#!/bin/bash

# Remove trace rules
nft delete chain ip nat trace_chain
nft delete chain ip filter4 trace_chain
nft delete chain ip6 filter6 trace_chain
echo "chain trace removed"
```

`chmod 750 /etc/network/remove_trace_nft`



## Trace usage

`nft monitor trace`