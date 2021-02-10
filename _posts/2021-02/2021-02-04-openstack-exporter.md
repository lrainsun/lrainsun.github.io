---
layout: post
title:  "openstack exporter启动失败"
date:   2021-02-04 23:00:00 +0800
categories: Prometheus
tags: Prometheus-Exporter
excerpt: openstack exporter启动失败
mathjax: true
typora-root-url: ../
---

# openstack exporter启动失败

我们在启用prometheus openstack exporter的时候，总是遇到这样的问题：

```shell
[root@edge-2-ocp-dev-sjc02-d ~]# curl http://edge-2.ocp-dev-sjc02-d.cloud.prv.webex.com:9198/metrics
An error has occurred while serving metrics:

collected metric "openstack_nova_agent_state" { label:<name:"adminState" value:"enabled" > label:<name:"disabledReason" value:"" > label:<name:"hostname" value:"ctrl-1.ocp-dev-sjc02-d.cloud.prv.webex.com" > label:<name:"id" value:"2" > label:<name:"service" value:"nova-conductor" > label:<name:"zone" value:"internal" > counter:<value:1 > } was collected before with the same name and label values
```

换了好多image也不管用，之前19年有过这个[bug](https://github.com/openstack-exporter/openstack-exporter/issues/37)，但是看起来，我们已经是新代码了，label里已经有ID，那为啥还有问题呢？

又仔细读了一下错误日志，跑到Openstack环境看了一下才知道，原来

![image-20210204225746822](/../assets/images/image-20210204225746822.png)

启用了cell之后，nova conductor有两个，一个是nova conductor，一个是nova super conductor，但是呢他们的名字又都叫nova-conductor

```
| 90a34d86-d651-4fe0-ba8a-2ef92aec275e | nova-conductor | ctrl-1.ocp-dev-sjc02-d.cloud.prv.webex.com | internal | enabled | up | 2021-02-04T09:14:49.000000 | - | False |
| 9e47adc8-8302-4d21-a26f-cad1b70b4fc1 | nova-conductor | ctrl-1.ocp-dev-sjc02-d.cloud.prv.webex.com | internal | enabled | up | 2021-02-04T09:14:50.000000 | - | False |
```

一个在cell0数据库里边，一个在cell1库里

而且就这么巧，ctrl-1上的nova-conductor跟nova-super-conductor连ID都是一样的

```shell
| 2021-02-03 03:04:23 | 2021-02-04 09:24:09 | NULL | 2 | ctrl-1.ocp-dev-sjc02-d.cloud.prv.webex.com | nova-conductor | conductor | 10916 | 0 | 0 | NULL | 2021-02
| 2021-02-03 03:05:23 | 2021-02-04 09:24:40 | NULL | 2 | ctrl-1.ocp-dev-sjc02-d.cloud.prv.webex.com | nova-conductor | conductor | 10914 | 0 | 0 | NULL | 2021-02-04 0
```