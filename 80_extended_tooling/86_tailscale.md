---
layout: default 
title: Tailscale
parent: Extended tooling
has_children: false
nav_order: 86
has_toc: false
---

# Tailscale

## Install

```bash
curl -fsSL https://pkgs.tailscale.com/stable/debian/bookworm.noarmor.gpg | tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null
curl -fsSL https://pkgs.tailscale.com/stable/debian/bookworm.tailscale-keyring.list | tee /etc/apt/sources.list.d/tailscale.list

apt update
apt install tailscale

tailscale up --auth-key <TS_AUTH_KEY> --netfilter-mode=off
```

{: .important }
> Replace `<TS_AUTH_KEY>` with the proper value.

{: .info }
> `--netfilter-mode=off` will prevent tailscale to update iptable/nftable rules.
