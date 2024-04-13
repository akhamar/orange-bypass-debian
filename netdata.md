# NetData

## Install

> mkdir /opt/netdata
>
> nano /opt/netdata/docker-compose.yml

```bash
services:
  netdata:
    image: netdata/netdata
    container_name: netdata
    #ports:
    #  - 80:19999/tcp
    pid: host
    network_mode: host
    restart: unless-stopped
    cap_add:
      - SYS_PTRACE
      - SYS_ADMIN
    security_opt:
      - apparmor:unconfined
    volumes:
      #- netdataconfig:/etc/netdata
      #- netdatalib:/var/lib/netdata
      #- netdatacache:/var/cache/netdata
      - ./netdataconfig:/etc/netdata
      - ./netdatalib:/var/lib/netdata
      - ./netdatacache:/var/cache/netdata
      - /etc/passwd:/host/etc/passwd:ro
      - /etc/group:/host/etc/group:ro
      - /etc/localtime:/etc/localtime:ro
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /etc/os-release:/host/etc/os-release:ro
      - /var/log:/host/var/log:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro

volumes:
  netdataconfig:
  netdatalib:
  netdatacache:
```

## Config

### Auto update

> crontab -e

```bash
# each two days
0 0 */2 * * cd /opt/netdata && docker compose pull && docker compose up --force-recreate --build -d > /var/log/netdata_update.log 2>&1
```

### netdata config

> nano /opt/netdata/netdataconfig/netdata.conf

```bash
[web]
default port = 80
```

> nano /opt/netdata/netdataconfig/go.d/prometheus.conf

```bash
jobs:
  - name: netfilter
    url: 'http://127.0.0.1:9630/metrics'

```




# NetData exporters

## nftable exporter

> mkdir /opt/netdata-exporters
>
> mkdir /opt/netdata-exporters/nftables

> cd /opt/netdata-exporters/nftables
>
> wget https://github.com/USA-RedDragon/nftables_exporter/releases/download/v0.2.0/nftables_exporter-linux-amd64
>
> mv /opt/netdata-exporters/nftables/nftables_exporter-linux-amd64 /opt/netdata-exporters/nftables/nftables_exporter
>
> chmod 750 /opt/netdata-exporters/nftables/nftables_exporter

> nano /opt/netdata-exporters/nftables/nftables_exporter.yaml

```bash
nftables_exporter:
  bind_to: "127.0.0.1:9630"
  url_path: "/metrics"
  nft_location: /sbin/nft
```

> nano /etc/systemd/system/nftables_exporter.service

```bash
[Unit]
Description=nftables exporter service
After=network-online.target

[Service]
Type=simple
PIDFile=/run/nftables_exporter.pid
ExecStart=/opt/netdata-exporters/nftables/nftables_exporter --config=/opt/netdata-exporters/nftables/nftables_exporter.yaml
User=root
Group=root
SyslogIdentifier=nftables_exporter
Restart=on-failure
RemainAfterExit=no
RestartSec=100ms
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

> systemctl enable nftables_exporter.service
>
> systemctl restart nftables_exporter.service