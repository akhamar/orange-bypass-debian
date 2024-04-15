---
layout: default 
title: DHCP Server
has_children: false
nav_order: 40
has_toc: false
---

# DHCP Server

## DHCP IPV4 & IPV6 Server binding

`nano /etc/default/isc-dhcp-server`

```bash
INTERFACESv4="lan"
INTERFACESv6="lan"
```

## DHCP IPV4 Server

`nano /etc/dhcp/dhcpd.conf`

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

{: .important }
> Replace placeholders value with proper values.

## DHCP IPV6 Server

`nano /etc/dhcp/dhcpd6.conf.template`

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
subnet6 ${IPV6_DELEGATION_64}::/64 {
  # Invite pool range
  range6 ${IPV6_DELEGATION_64}::aaaa ${IPV6_DELEGATION_64}::bbbb;

  # DNS
  option dhcp6.name-servers ${IPV6_DELEGATION_64}::dddd;
}

ddns-update-style none;


##################
#### Lease

# XXXXXX
host XXXXXX {
  host-identifier option dhcp6.client-id xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx;
  fixed-address6 ${IPV6_DELEGATION_64}::eeee;
}

# YYYYYY
host YYYYYY {
  host-identifier option dhcp6.client-id xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx;
  fixed-address6 ${IPV6_DELEGATION_64}::ffff;
}

...
```

{: .important }
> Replace placeholders value with proper values.