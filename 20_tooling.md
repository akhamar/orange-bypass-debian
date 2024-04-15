---
layout: default 
title: Tooling & drivers
has_children: false
nav_order: 20
has_toc: false
---

# Tooling & drivers

## Necessary packages

`apt install htop rsyslog ethtool tcpdump conntrack net-tools iproute2 vlan cgroup-tools isc-dhcp-server radvd`

## Plymouth (Optional)

On debian UEFI and only console (no UI), it seem plymouth is not installed. Plymouth allow to show Starting service [OK] / [Failed] on startup.

It an usefull package to see what's going on on startup.

`apt install plymouth`

## Oh-My-Zsh (Optional)

`apt install git curl zsh`

`sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"`