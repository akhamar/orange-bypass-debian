---
layout: default 
title: VerNum activated on startup
parent: Extended tooling
has_children: false
nav_order: 82
has_toc: false
---

# VerNum activated on startup (optional)

`nano /usr/local/bin/numlock`

```bash
#!/bin/bash

for tty in /dev/tty{1..6}
do
    /usr/bin/setleds -D +num < "$tty";
done
```

`chmod 755 /usr/local/bin/numlock`

`nano /etc/systemd/system/numlock.service`

```bash
[Unit]
Description=numlock

[Service]
ExecStart=/usr/local/bin/numlock
StandardInput=tty
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

`systemctl enable numlock.service`