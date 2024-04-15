---
layout: default 
title: RADVD
has_children: false
nav_order: 50
has_toc: false
---

# RADVD

`nano /etc/radvd.conf`

```bash
# LAN
interface lan {

        # A flag indicating whether or not the router sends periodic router advertisements and responds to router solicitations.
        AdvSendAdvert on;
        # The minimum time allowed between sending unsolicited multicast router advertisements
        MinRtrAdvInterval 200;
        # The maximum time allowed between sending unsolicited multicast router advertisements
        MaxRtrAdvInterval 600;
        # MTU
        AdvLinkMTU 1500;
        #AdvLinkMTU 9000;
        # The preference associated with the default router
        AdvDefaultPreference medium;
        # Hosts use the administered (stateful) protocol for address autoconfiguration
        AdvManagedFlag on;
        # Hosts use the administered (stateful) protocol for autoconfiguration of other (non-address) information
        AdvOtherConfigFlag on;

        prefix ::/64 {
                DeprecatePrefix on;
                # When set, indicates that this prefix can be used for on-link determination
                AdvOnLink on;
                # When set, indicates that this prefix can be used for autonomous address configuration as specified in RFC 4862.
                AdvAutonomous off;
        };

        # Broadcast DNS server
        RDNSS xxxx:xxxx:xxxx::xxxx xxxx:xxxx:xxxx::xxxx {
        };

        # Broadcast domain
        DNSSL aaaa.bbbb.cc {
        };

};
```

{: .important }
> Replace placeholders value with proper values

{: .info }
> Both *RDNSS* and *DNSSL* are optional.
>
> You might not want or need them. Remove at your discretion.