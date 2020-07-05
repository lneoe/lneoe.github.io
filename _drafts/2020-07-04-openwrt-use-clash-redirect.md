---
layout: post
title: openwrt use clash redirect
date: 2020-07-04 18:38 +0800
categories: network
tags: clash
comments: true
---

`openwrt` use `clash transparent proxy`

#### clash config

```json
# enable redirect
redir-port: 7892
```

#### set iptables 

###### edit `/etc/firewall.user`

```bash
# edit /etc/firewall.user
# Create CLASH chain
iptables -t nat -N CLASH

# Bypass private IP address ranges
iptables -t nat -A CLASH -d 10.0.0.0/8 -j RETURN
iptables -t nat -A CLASH -d 127.0.0.0/8 -j RETURN
iptables -t nat -A CLASH -d 169.254.0.0/16 -j RETURN
iptables -t nat -A CLASH -d 172.16.0.0/12 -j RETURN
iptables -t nat -A CLASH -d 192.168.0.0/16 -j RETURN
iptables -t nat -A CLASH -d 224.0.0.0/4 -j RETURN
iptables -t nat -A CLASH -d 240.0.0.0/4 -j RETURN
iptables -t nat -A CLASH -d ${your_server_ip} -j RETURN

# Disable the proxy for some ip
iptables -t nat -A CLASH -s ${some-ip} -j RETURN

# Redirect all TCP traffic to 7892 port, where Clash listens
iptables -t nat -A CLASH -p tcp -j REDIRECT --to-ports 7892
iptables -t nat -A PREROUTING -p tcp -j CLASH
```

###### create `/etc/init.d/clash`
```bash
$ cat > /etc/init.d/clash << EOF
#!/bin/sh /etc/rc.common

START=90

USE_PROCD=1

start_service() {
        procd_open_instance
        procd_set_param command /usr/bin/clash -d /etc/clash
        procd_set_param respawn 300 0 5 # threshold, timeout, retry
        procd_set_param file /etc/clash/config.yaml
        procd_set_param stdout 1
        procd_set_param stderr 1
        procd_set_param pidfile /var/run/clash.pid
        procd_close_instance
}
EOF
$ chmod +x /etc/init.d/clash
# start clash, and read log
$ /etc/init.d/clash start && logread -e clash -f
# check every thing is right
$ /etc/init.d/clash enable
```

###### reference
[clash config](https://github.com/Dreamacro/clash/wiki/configuration#all-configuration-options)
[/etc/init.d/clash](https://blog.birkhoff.me/running-clash-on-openwrt-as-a-transparent-proxy)

