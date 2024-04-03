> [!warning]
> All those steps have only been tested on Debian 12
>
> You might need to adapt on other distribution or version

# Interface rename

## WAN

> nano /etc/systemd/network/10-wan.link

```bash
[Match]
MACAddress=xx:xx:xx:xx:xx:xx

[Link]
Name=wan
```
where `xx:xx:xx:xx:xx:xx` correspond to the MAC addr of WAN interface


## LAN

> nano /etc/systemd/network/10-lan.link

```bash
[Match]
MACAddress=xx:xx:xx:xx:xx:xx

[Link]
Name=lan
```
where `xx:xx:xx:xx:xx:xx` correspond to the MAC addr of LAN interface



# Tooling & drivers

## Necessary package

> apt install git curl zsh htop ethtool tcpdump net-tools iproute2 vlan cgroup-tools isc-dhcp-server radvd
>
> sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"


## BMC (optional)

> [!important] 
> All commandes are necessary

> apt install firmware-bnx2x dkms
>
> curl https://raw.githubusercontent.com/JAMESMTL/snippets/master/bnx2x/patches/dkms-init.sh | sudo sh | sudo tee /usr/src/dkms-init.log
>
> curl https://raw.githubusercontent.com/JAMESMTL/snippets/master/bnx2x/patches/dkms-update.sh | sudo sh | sudo tee /usr/src/dkms-update.log
>
> apt remove firmware-bnx2x
>
> curl https://raw.githubusercontent.com/JAMESMTL/snippets/master/bnx2x/patches/dkms-init.sh | sudo sh | sudo tee /usr/src/dkms-init.log
>
> apt install firmware-bnx2x



# Interface configuration

## LAN

> nano /etc/network/interfaces.d/lan

```bash
# LAN
allow-hotplug lan
iface lan inet static
        address 192.168.1.1/24
```

## WAN

### DHCP

> nano /etc/dhcp/dhclient-orange-v4.conf

```bash
# Definition
option user-class       code 77 = string;
option authentication   code 90 = string;

# Send Option
send dhcp-parameter-request-list 1, 3, 6, 15, 28, 51, 58, 59, 90, 119, 120, 125;
# Uncomment for absolutly no Orange domain/dns/server-name related
#send dhcp-parameter-request-list 1, 3, 28, 51, 58, 59, 90, 120, 125;

send vendor-class-identifier "sagem";
send dhcp-client-identifier = hardware;                                 # Equivalent to 01:xx:xx:xx:xx:xx:xx but use mac specified on interface (cf. hw-mac-address on iface)
send user-class "+FSVDSL_livebox.Internet.softathome.Livebox5";
send authentication 00:00:00:00:00:00:00:00:00:00:00:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx;

# Uncomment to avoid getting Orange DNS servers
#supersede domain-name "aaaa.bbbb.cc";
#supersede domain-search "aaaa.bbbb.cc";
#supersede domain-name-servers xxx.xxx.xxx.xxx;
```

> nano /etc/dhcp/dhclient-orange-v6.conf

```bash
# Definition
option dhcp6.vendorclass        code 16 = string;
option dhcp6.userclass          code 15 = string;
option dhcp6.auth               code 11 = string;

# Send Option
# 73:61:67:65:6d == sagem
send dhcp6.vendorclass 00:00:04:0e:00:05:73:61:67:65:6d;
# 2b:46:53:56:44:53:4c:5f:6c:69:76:65:62:6f:78:2e:49:6e:74:65:72:6e:65:74:2e:73:6f:66:74:61:74:68:6f:6d:65:2e:4c:69:76:65:62:6f:78:35 == "+FSVDSL_livebox.Internet.softathome.Livebox5"
send dhcp6.userclass 00:2b:46:53:56:44:53:4c:5f:6c:69:76:65:62:6f:78:2e:49:6e:74:65:72:6e:65:74:2e:73:6f:66:74:61:74:68:6f:6d:65:2e:4c:69:76:65:62:6f:78:35;
# auth str
send dhcp6.auth 00:00:00:00:00:00:00:00:00:00:00:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx;

request dhcp6.auth, dhcp6.vendorclass, dhcp6.userclass;
```

> nano /etc/dhcp/dhclient-enter-hooks.d/ipv6-internal

```bash
#!/bin/bash

RUN="yes"

if [ "$RUN" = "yes" ] && [ -n "$new_ip6_prefix" ] && [ -n "$interface_internal" ]
then
        prefix="${new_ip6_prefix%00::/*}"
        for interface_index in $interface_internal
        do
                interface="${interface_index%:*}"
                index="${interface_index#*:}"

                echo "========================="
                echo "| interface: $interface"
                echo "========================="
                echo "| prefix: $prefix"
                echo "| index:  $index"
                echo "| ipv6:   $prefix$index::1/64"
                echo "========================="
                echo ""

                ip -family inet6 address flush dev "$interface" scope global
                ip -family inet6 address add "$prefix$index::1/64" dev "$interface"
        done
fi
```


### Interface

> nano /etc/network/wait_for_wan

```bash
#!/bin/bash

TIMEOUT=120

until ip addr show wan | grep -q "state UP"
do
        sleep 2
        if [[ $SECONDS -ge $TIMEOUT ]]
        then
                echo "Timeout waiting for ONU module to be UP"
                exit 1
        fi
done

```

> nano /etc/network/interfaces.d/wan

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
        up /etc/network/wait_for_wan

        # Egress prio 6:6
        up ip l s $IFACE type vlan egress 6:6

        # CGroup Up
        up mkdir -p /sys/fs/cgroup/net_prio && mount -t cgroup -o net_prio net_prio /sys/fs/cgroup/net_prio 2>/dev/null || true
        up cgcreate -g net_prio:dhcp-orange
        up cgset -r "net_prio.ifpriomap=$IFACE 6" dhcp-orange

        # DHCP Up
        up cgexec -g net_prio:dhcp-orange --sticky -- dhclient -4 -i $IFACE -cf /etc/dhcp/dhclient-orange-v4.conf -df /var/lib/dhcp/dhclient-orange-v4.duid -lf /var/lib/dhcp/dhclient-orange-v4.lease -v
        up sleep 2
        up cgexec -g net_prio:dhcp-orange --sticky -- dhclient -6 -P -D LL -i $IFACE -cf /etc/dhcp/dhclient-orange-v6.conf -df /var/lib/dhcp/dhclient-orange-v6.duid -lf /var/lib/dhcp/dhclient-orange-v6.lease -e interface_internal="$IFACE:01 lan:02" -v

        # DHCP Down
        down cgexec -g net_prio:dhcp-orange --sticky -- dhclient -6 -P -D LL -i $IFACE -cf /etc/dhcp/dhclient-orange-v6.conf -df /var/lib/dhcp/dhclient-orange-v6.duid -lf /var/lib/dhcp/dhclient-orange-v6.lease -e interface_internal="$IFACE:01 lan:02" -v -r
        down sleep 2
        down cgexec -g net_prio:dhcp-orange --sticky -- dhclient -4 -i $IFACE -cf /etc/dhcp/dhclient-orange-v4.conf -df /var/lib/dhcp/dhclient-orange-v4.duid -lf /var/lib/dhcp/dhclient-orange-v4.lease -v -r

        # CGroup down
        down cgdelete -g net_prio:dhcp-orange
        down umount /sys/fs/cgroup/net_prio 2>/dev/null || true
```


# Serveur DHCP

> apt install isc-dhcp-server

> nano /etc/default/isc-dhcp-server

```bash
INTERFACESv4="lan"
INTERFACESv6="lan"
```

## Serveur DHCP IPV4

> nano /etc/dhcp/dhcpd.conf

```bash
##################
#### Config

# Global config
option domain-name "aaaa.bbbb.cc";
option ldap-server code 95 = text;
option arch code 93 = unsigned integer 16; # RFC4578
option pac-webui code 252 = text;
# MTU flag
option mtu-flag code 26 = unsigned integer 16;

# Lease config
default-lease-time 7200;
max-lease-time 86400;
log-facility local7;
one-lease-per-client true;
deny duplicates;
ping-check true;
update-conflict-detection false;
authoritative;

# Subnet
subnet 192.168.1.0 netmask 255.255.255.0 {
  # Invite pool range
  pool {
    option domain-name-servers 192.168.1.xxx;
    range 192.168.1.xxx 192.168.1.xxx;
  }
  option routers 192.168.1.1;
  option domain-name "aaaa.bbbb.cc";
  option domain-search "aaaa.bbbb.cc";
  option domain-name-servers 192.168.1.xxx;

  # MTU 9000 (optional)
  #option mtu-flag 9000;
}


##################
#### Lease

# XXXXXXX
host XXXXXXX {
  hardware ethernet xx:xx:xx:xx:xx:xx;
  fixed-address 192.168.1.XXX;
  option host-name "XXXXXXX";
  set hostname-override = config-option host-name;
}

# YYYYYYY
host YYYYYYY {
  hardware ethernet xx:xx:xx:xx:xx:xx;
  option dhcp-client-identifier "YYYYYYY";
  fixed-address 192.168.1.XXX;
  option host-name "YYYYYYY";
  set hostname-override = config-option host-name;
}

...
```
> replace xxx with proper values

## Serveur DHCP IPV6 & Radvd

> nano /etc/dhcp/dhcpd6.conf

```bash
##################
#### Config

# Global config
option dhcp6.domain-search "aaaa.bbbb.cc";
option dhcp6.rapid-commit;

# Lease config
default-lease-time 7200;
max-lease-time 86400;
log-facility local7;
one-lease-per-client true;
deny duplicates;
ping-check true;
update-conflict-detection false;
authoritative;

# Subnet
subnet6 xxxx:xxxx:xxxx:xxxx::/64 {
  # Invite pool range
  range6 xxxx:xxxx:xxxx:xxxx::aaaa xxxx:xxxx:xxxx:xxxx::bbbb;

  # DNS
  option dhcp6.name-servers xxxx:xxxx:xxxx:xxxx::dddd;
}

ddns-update-style none;


##################
#### Lease

# XXXXXX
host XXXXXX {
  host-identifier option dhcp6.client-id xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx;
  fixed-address6 xxxx:xxxx:xxxx:xxxx::eeee;
}

# YYYYYY
host YYYYYY {
  host-identifier option dhcp6.client-id xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx;
  fixed-address6 xxxx:xxxx:xxxx:xxxx::ffff;
}

...
```
> replace xxx with proper values


> nano /etc/radvd.conf

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

}
```



## Tooling

Ajout de la base de données des manufacturer
> wget http://standards-oui.ieee.org/oui.txt
>
> mv oui.txt /usr/local/etc/

Connnaitre les leases actif
> dhcp-lease-list

# Routing

## Enable nftables

> systemctl enable nftables.service

## Configure nftables

> nano /etc/nftables.conf

```bash
#!/usr/sbin/nft -f

flush ruleset

define LAN_IPV4_SUBNET  = { 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 }
define LAN_IPV6_SUBNET  = { xxx:xxx:xxx:xxxx::/64 }
define LAN_SSH_PORT     = { yyy }

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
        flowtable fastpath {
                hook ingress priority 0; devices = { lan, vlan832 };
        }
        chain input {
                type filter hook input priority filter; policy drop;
                iif lo accept
                ct state { related, established } accept
                iifname "lan" ip saddr $LAN_IPV4_SUBNET accept                  # From LAN to WAN
        }
        chain forward {
                type filter hook forward priority filter; policy drop;
                ct state { related, established } meta l4proto { tcp, udp } flow offload @fastpath
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
        flowtable fastpath {
                hook ingress priority 0; devices = { lan, vlan832 };
        }
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
                ct state { related, established } meta l4proto { tcp, udp } flow offload @fastpath
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

> [!warning]
> Adding fastpath on nftable explicitely declare lan and vlan832
>
> By doing that the service start up could crash when the interface vlan832 is not up yet
>
> To mitigate this issue, you can update the systemd service for nftable
>
> Adding `Restart=on-failure` and `RestartSec=30` in the section `[Service]` will solve this issue

> nano /etc/systemd/system/sysinit.target.wants/nftables.service

```bash
[Unit]
Description=nftables
Documentation=man:nft(8) http://wiki.nftables.org
Wants=network-pre.target
Before=network-pre.target shutdown.target
Conflicts=shutdown.target
DefaultDependencies=no

[Service]
Type=oneshot
RemainAfterExit=yes
StandardInput=null
ProtectSystem=full
ProtectHome=true
ExecStart=/usr/sbin/nft -f /etc/nftables.conf
ExecReload=/usr/sbin/nft -f /etc/nftables.conf
ExecStop=/usr/sbin/nft flush ruleset
Restart=on-failure              <<<<<<<<<
RestartSec=30                   <<<<<<<<<


[Install]
WantedBy=sysinit.target
```

> systemctl daemon-reload
> 
> systemctl restart nftables.service


## IP forward

> nano /etc/sysctl.conf

```
# IP forwarding (IPV4 / IPV6)
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1

# Desactivate accepting Router Advertisements
net.ipv6.conf.lan.accept_ra=0

# Accept accepting Router Advertisements even when 'net.ipv6.conf.all.forwarding=1' (force Solicitation/Advertisements true)
net.ipv6.conf.default.accept_ra=2
```

## Test

> ifdown vlan832 && ifup vlan832
>
> systemctl restart isc-dhcp-server.service
>
> systemctl restart radvd.service
>
> systemctl restart nftables.service


# Dynamic DNS with DDClient

## Install

> apt install ddclient dh-autoreconf

> [!important] 
> Clone the lastest version of ddclient if the current version is not at least 3.11.3_0
> You don't need to clone if ddclient is more up to date

> [!note]
> Only needed if ddclient is not at least 3.11.3_0

> cd /tmp && git clone https://github.com/ddclient/ddclient.git && cd /tmp/ddclient
>
> ./autogen
>
> ./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var
>
> make
>
> make install

## Configuration

> nano /etc/ddclient.conf

> [!note]
> Example with gandi
```
ssl=yes
force=yes

protocol=gandi
zone=bbbb.cc
password=<PASSWORD>
usev4=ifv4
usev6=ifv6
ifv4=vlan832
ifv6=vlan832
aaaa.bbbb.cc
```
