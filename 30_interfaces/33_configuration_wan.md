---
layout: default 
title: Interfaces configuration (WAN)
parent: Interfaces
has_children: false
nav_order: 33
has_toc: false
---

# Interfaces configuration (WAN)

## DHCP Client

### DHCP Orange option generator

`nano /etc/dhcp/dhclient-orange-generator`

```bash
#!/bin/bash

LOGIN='fti/XXXXXXX'
PASSWORD='YYYYYYY'
LIVEBOX_VERSION=5
LIVEBOX_HARDWARE='sagem'

# Forge DHCP option 90
tohex() {
  for h in $(echo $1 | sed "s/\(.\)/\1 /g"); do printf %02x \'$h; done
}

addsep() {
  echo $(echo $1 | sed "s/\(.\)\(.\)/:\1\2/g")
}

r=$(dd if=/dev/urandom bs=1k count=1 2>&1 | md5sum | cut -c1-16)
id=${r:0:1}
h=3c12$(tohex ${r})0313$(tohex ${id})$(echo -n ${id}${PASSWORD}${r} | md5sum | cut -c1-32)

# vendor class
export VENDOR_CLASS_IDENTIFIER_4=${LIVEBOX_HARDWARE}
export VENDOR_CLASS_IDENTIFIER_6=00:00:04:0e:00:05$(addsep $(tohex ${LIVEBOX_HARDWARE}))
echo "Vendor class has been generated"

# user class
export USER_CLASS_4=+FSVDSL_livebox.Internet.softathome.Livebox${LIVEBOX_VERSION}
export USER_CLASS_6=00$(addsep $(tohex "+FSVDSL_livebox.Internet.softathome.Livebox${LIVEBOX_VERSION}"))
echo "User class has been generated"

# option 90
export AUTHENTICATION_STR=00:00:00:00:00:00:00:00:00:00:00:1a:09:00:00:05:58:01:03:41:01:0d$(addsep $(tohex ${LOGIN})${h})
echo "Option 90 has been generated"

# Generate DHCP client (ivp4 and ivp6) files
envsubst < /etc/dhcp/dhclient-orange-v4.conf.template > /etc/dhcp/dhclient-orange-v4.conf
envsubst < /etc/dhcp/dhclient-orange-v6.conf.template > /etc/dhcp/dhclient-orange-v6.conf
```

{: .important }
> Replace `XXXXXXX`, `YYYYYYY`, `Livebox version` and `Livebox hardware` with the proper values.

`chmod 750 /etc/dhcp/dhclient-orange-generator`

### DHCP Client IPV4

`nano /etc/dhcp/dhclient-orange-v4.conf.template`

```bash
# Debug: tcpdump -i wan port 67 or port 68 -e -n -v

# Definition
option user-class       code 77 = string;
option authentication   code 90 = string;

# Send Option
send dhcp-parameter-request-list 1, 3, 6, 15, 28, 51, 58, 59, 90, 119, 120, 125;
# Uncomment for absolutly no Orange domain/dns/server-name related
#send dhcp-parameter-request-list 1, 3, 28, 51, 58, 59, 90, 120, 125;
send vendor-class-identifier "${VENDOR_CLASS_IDENTIFIER_4}";
send dhcp-client-identifier = hardware;                                 # equivalent to 01:xx:xx:xx:xx:xx:xx but use mac specified on interface (cf. hw-mac-address on iface)
send user-class "${USER_CLASS_4}";
send authentication ${AUTHENTICATION_STR};

# Uncomment to avoid getting Orange DNS servers
#supersede domain-name "aaaa.bbbb.cc";
#supersede domain-search "aaaa.bbbb.cc";
#supersede domain-name-servers xxx.xxx.xxx.xxx;
```

### DHCP Client IPV6

`nano /etc/dhcp/dhclient-orange-v6.conf.template`

```bash
# Debug: tcpdump -i wan port 546 or port 547 -e -n -v

# Definition
option dhcp6.vendorclass        code 16 = string;
option dhcp6.userclass          code 15 = string;
option dhcp6.auth               code 11 = string;

# Send Option
# 73:61:67:65:6d == sagem
send dhcp6.vendorclass ${VENDOR_CLASS_IDENTIFIER_6};
# 2b:46:53:56:44:53:4c:5f:6c:69:76:65:62:6f:78:2e:49:6e:74:65:72:6e:65:74:2e:73:6f:66:74:61:74:68:6f:6d:65:2e:4c:69:76:65:62:6f:78:35 == "+FSVDSL_livebox.Internet.softathome.Livebox5"
send dhcp6.userclass ${USER_CLASS_6};
# auth str
send dhcp6.auth $AUTHENTICATION_STR;

request dhcp6.auth, dhcp6.vendorclass, dhcp6.userclass;
```

### DHCP Client IPV6 Subnet allocation

`nano /etc/dhcp/dhclient-enter-hooks.d/ipv6-internal`

```bash
#!/bin/bash

RUN="yes"

if [ "$RUN" = "yes" ] && [ -n "$new_ip6_prefix" ] && [ -n "$interface_internal" ]
then

        # Flush all ip on interface
        interface_selection=$(echo "$interface_internal" | awk '{print $2}' | awk -F ':' '{print $1}')
        ip -family inet6 address flush dev "$interface_selection" scope global

        # Compute prefix
        prefix="${new_ip6_prefix%00::/*}"
        for interface_index in $interface_internal
        do
                interface="${interface_index%:*}"
                index="${interface_index#*:}"

                echo ""
                echo "========================="
                echo "| interface: $interface"
                echo "========================="
                echo "| prefix: $prefix"
                echo "| index:  $index"
                echo "| ipv6:   $prefix$index::1/64"
                echo "========================="

                ip -family inet6 address add "$prefix$index::1/64" dev "$interface"

                # Automatically associate a subnet6 corresponding to lan IVP6 address subnet
                # DHCPD6
                if [ "$interface" = "lan" ] && [ "$index" = "02" ] && [ -f /etc/dhcp/dhcpd6.conf.template ]
                then
                        export IPV6_DELEGATION_64=$prefix$index
                        export IPV6_DELEGATION_56=$prefix
                        echo ""
                        echo "*************************"
                        echo "* Configuring isch-dhcp-server IPV6 delegation for subnet6=${IPV6_DELEGATION_56}${index}::/64"
                        envsubst < /etc/dhcp/dhcpd6.conf.template > /etc/dhcp/dhcpd6.conf
                fi

                # RADVD
                if [ "$interface" = "lan" ] && [ "$index" = "03" ] && [ -f /etc/radvd.conf.template ]
                then
                        export IPV6_DELEGATION_64=$prefix$index
                        export IPV6_DELEGATION_56=$prefix
                        echo ""
                        echo "*************************"
                        echo "* Configuring radvd SLAAC IPV6 delegation for subnet6=${IPV6_DELEGATION_56}XX::/64"
                        envsubst < /etc/radvd.conf.template > /etc/radvd.conf
                fi
        done
fi
```

`chmod 750 /etc/dhcp/dhclient-enter-hooks.d/ipv6-internal`


## Interface

### Wait for ONU Up link

{: .info }
> There is two version of the `wait for wan` script. The first version is for the ONU inserted directly in the NIC of the router. The second version is when you put a switch in between the router and the ONU.

#### First version - ONU directly inserted in the NIC

`nano /etc/network/wait_for_wan`

```bash
#!/bin/bash

IFACE=$1
TIMEOUT=$2

if [[ -z $IFACE ]]
then
        echo "IFACE need to be set"
        exit 1
fi

if [[ -z $TIMEOUT ]]
then
        TIMEOUT=180
fi

# We wait until 'Laser output power' is greater than 0
echo "Waiting for ONU to be UP and running"
until [[ $(ethtool -m $IFACE | grep -E 'Laser output power\s+:' | awk '{print $(NF - 1)}' | cut -d'.' -f1) -gt 0 ]]
do
        sleep 2
        if [[ $SECONDS -ge $TIMEOUT ]]
        then
                echo "Timeout ($TIMEOUT second) waiting for ONU module to be UP"
                exit 1
        fi
done
echo "ONU UP and running"
```

`chmod 750 /etc/network/wait_for_wan`

#### Second version - ONU inserted in a switch with a link between the switch and the NIC

`nano /etc/network/wait_for_wan`

```bash
#!/bin/bash

IFACE=$1
TIMEOUT=$2

if [[ -z $IFACE ]]
then
        echo "IFACE need to be set"
        exit 1
fi

if [[ -z $TIMEOUT ]]
then
        TIMEOUT=180
fi

# We wait until the IFACE show as UP
echo "Waiting for ONU to be UP and running"
until ip addr show $IFACE | grep -q "state UP"
do
        sleep 2
        if [[ $SECONDS -ge $TIMEOUT ]]
        then
                echo "Timeout ($TIMEOUT second) waiting for ONU module to be UP"
                exit 1
        fi
done
echo "ONU UP and running"
```

`chmod 750 /etc/network/wait_for_wan`

### Handling COS6 (802.1Q prio 6) and DSCP cs6 for Orange

`nano /etc/network/inject_pcp_6`

```bash
#!/bin/bash

# Add PCP 6 (802.1Q prio 6)
nft add "table netdev filter"
nft add "chain netdev filter egress { type filter hook egress device $IFACE priority 0; }"

# IPV4
nft insert "rule netdev filter egress udp dport 67 meta priority set 0:6 ip dscp set cs6 comment \"Set CoS value to 6 for DHCPv4 packets\""
nft insert "rule netdev filter egress ether type arp meta priority set 0:6 comment \"Set CoS value to 6 for arp packets\""

# IPV6
nft insert "rule netdev filter egress udp dport 547 meta priority set 0:6 ip6 dscp set cs6 comment \"Set CoS value to 6 for DHCPv6 packets\""
nft insert "rule netdev filter egress icmpv6 type { nd-router-solicit, nd-neighbor-solicit, nd-neighbor-advert } meta priority set 0:6 ip6 dscp set cs6 comment \"Set CoS value to 6 for RS/NS/NA packets\""

echo "Injecting PCP 6 / 802.1Q prio 6 on egress DHCP packet [iface: $IFACE]"
```

### Interface configuration

`nano /etc/network/interfaces.d/wan`

```bash
# WAN
auto wan
allow-hotplug wan
iface wan inet manual

# VLAN832
auto vlan832
allow-hotplug vlan832
iface vlan832 inet manual

        # Bind vlan
        vlan-raw-device wan

        # LiveBox mac address
        hw-mac-address xx:xx:xx:xx:xx:xx

        # Wait for ONU to be UP
        up /etc/network/wait_for_wan wan

        # Reload NFTable
        up nft -f /etc/nftables.conf

        # Add nftable PCP 6 / 802.1Q prio 6 on egress DHCPv4/v6 packet
        up /etc/network/inject_pcp_6

        # Add nftable fastpath rules
        # Uncomment if you want to use flowtable fastpath
        #up /etc/network/inject_flowtable_fastpath

        # Generate Orange Options (user-class, vendor-class, option 90)
        up /etc/dhcp/dhclient-orange-generator

        # Egress prio 6:6 (aka VLAN-PCP)
        up ip l s $IFACE type vlan egress 6:6
        up sleep 2

        # DHCP Up
        up dhclient -4 -i $IFACE -cf /etc/dhcp/dhclient-orange-v4.conf -df /var/lib/dhcp/dhclient-orange-v4.duid -lf /var/lib/dhcp/dhclient-orange-v4.lease -v
        up sleep 2
        up dhclient -6 -P -D LL -i $IFACE -cf /etc/dhcp/dhclient-orange-v6.conf -df /var/lib/dhcp/dhclient-orange-v6.duid -lf /var/lib/dhcp/dhclient-orange-v6.lease -e interface_internal="$IFACE:01 lan:02 lan:03" -v

        # DHCP Down
        down dhclient -6 -P -D LL -i $IFACE -cf /etc/dhcp/dhclient-orange-v6.conf -df /var/lib/dhcp/dhclient-orange-v6.duid -lf /var/lib/dhcp/dhclient-orange-v6.lease -e interface_internal="$IFACE:01 lan:02 lan:03" -v -r
        down sleep 2
        down dhclient -4 -i $IFACE -cf /etc/dhcp/dhclient-orange-v4.conf -df /var/lib/dhcp/dhclient-orange-v4.duid -lf /var/lib/dhcp/dhclient-orange-v4.lease -v -r
```

{: .important }
> Replace `xx:xx:xx:xx:xx:xx` with the Livebox mac address.

{: .info }
> Uncomment `#up /etc/network/inject_flowtable_fastpath` if you want to use flowtable fastpath.
