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

