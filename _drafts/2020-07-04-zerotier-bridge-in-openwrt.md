---
layout: post
title: zerotier bridge in openwork
date: 2020-07-04 18:38 +0800
categories: network
tags: zerotier
comments: true
---


`openwrt` 使用 `zerotier bridge` 打通异地网络, 以下内容假设你已经注册了 `zerotier` 账号并创建  `zerotier network`.

#### 安装并设置 `zerotier`

```bash
$ opkg update
$ opkg install zerotier
## more trikers..
# set zerotier config_path
$ uci set zerotier.${replace_with_your_network}.config_path="/etc/zerotier-one"
# copy zerotier config
$ cp -r /tmp/lib/zerotier-one_${replace_with_your_network}/ /etc/zerotier-one
# reboot router ensure zerotier setting is right
$ reboot
```

[zerotier-openwrt#install](https://github.com/mwarning/zerotier-openwrt/wiki#installation)

[zerotier-openwrt#Preparing your ZeroTier network](https://zerotier.atlassian.net/wiki/spaces/SD/pages/7438339/Layer+2+Bridging+with+LEDE+OpenWRT)

#### 路由器中设置 `bridge`

[zerotier-openwrt#ConfigureOnYourRouter](https://zerotier.atlassian.net/wiki/spaces/SD/pages/7438339/Layer+2+Bridging+with+LEDE+OpenWRT)

