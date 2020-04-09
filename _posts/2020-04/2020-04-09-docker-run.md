---
layout: post
title:  "docker run"
date:   2020-04-09 23:00:00 +0800
categories: Docker
tags: Docker-Service
excerpt: docker run
mathjax: true
---

# docker run

嗯，今天跑docker run的时候，错误地把option放到后面去，被当做需要在容器内部执行的command了

导致option没有生效。。。

**docker run ：**创建一个新的容器并运行一个命令

### 语法

```shell
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

```shell
docker run -itd --privileged -p 8081:8081 -v /root/distributed-ocp-exporter/config:/config -h `hostname` -v /var/lib/nova/instances:/var/lib/nova/instances --name distributed_ocp_exporter -v=/run/dbus/system_bus_socket:/run/dbus/system_bus_socket -v=/proc:/host/proc:ro -v=/run/systemd:/run/systemd:ro --net=host --path.procfs=/host/proc --restart=always registry-qa.webex.com/ocp/distributed-ocp-exporter:v1.0.6 
```

