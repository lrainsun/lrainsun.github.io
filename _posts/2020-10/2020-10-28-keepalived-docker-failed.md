---
layout: post
title:  "keepalived container启动失败"
date:   2020-10-28 23:00:00 +0800
categories: Docker
tags: Docker-Service
excerpt: keepalived container启动失败
mathjax: true
---

# keepalived container启动失败

有碰到这样一个情况，当reboot host之后，keepalived container会启动失败，报错

```shell
daemon is already running
```

这是因为keepalived容器内的keepalived.pid文件在keepalived容器非正常退出时，没有正确删除，造成第二次启动时容器检查到pid文件已经存在，认为该进程已经存在，因为keepalived容器的启动检查机制只允许同一台主机上启动一个进程，所以无法启动

解决办法也很简单

修改Dockerfile如下

```dockerfile
FROM alpine:latest

RUN apk add --no-cache keepalived socat

RUN rm -rf /var/run/keepalived.pid
ENTRYPOINT ["/usr/sbin/keepalived"]
CMD ["-nld"]
```

可以重新构建keepalived镜像，在启动keepalived之前删除一遍keepalived.pid文件即可