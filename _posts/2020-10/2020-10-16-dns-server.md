---
layout: post
title:  "容器化的dns服务器"
date:   2020-10-16 23:01:00 +0800
categories: Linux
tags: Linux-Service
excerpt: 容器化的dns服务器
mathjax: true
typora-root-url: ../
---

# 容器化的dns服务器

对于私有的小型的dns服务器，我们可以采用容器化的dnsmasq来代替，很轻便

可以这样启动

```shell
docker run \
        --name dnsmasq \
        -d \
        -p 53:53/udp \
        -p 5380:8080 \
        -v /root/dnsmasq/dnsmasq.conf:/etc/dnsmasq.conf \
        -v /etc/hosts:/etc/hosts \
        -e "HTTP_USER=admin" \
        -e "HTTP_PASS=password" \
        --restart always \
        jpillora/dnsmasq
```

可以把dns记录配置在dnsmasq.conf里

```shell
#dnsmasq config, for a complete example, see:
#  http://oss.segetech.com/intra/srv/dnsmasq.conf
#log all dns queries
log-queries
#dont use hosts nameservers
no-resolv
#use cloudflare as default nameservers, prefer 1^4
server=10.240.240.1
server=10.240.241.1
strict-order
#serve all .company queries using a specific nameserver
server=/ocp.webex.com/192.168.0.4
#explicitly define host-ip mappings
address=/ocp-dev-sjc02-a-ctr-1.ocp.webex.com/192.168.0.11
address=/ocp-dev-sjc02-a-ctr-2.ocp.webex.com/192.168.0.12
address=/ocp-dev-sjc02-a-ctr-3.ocp.webex.com/192.168.0.13
```

或者配置在host的/etc/hosts里，都是可以生效的