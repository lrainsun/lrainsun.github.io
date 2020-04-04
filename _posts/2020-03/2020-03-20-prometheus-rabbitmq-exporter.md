---
layout: post
title:  "rabbitmq exporter not up"
date:   2020-03-20 23:00:00 +0800
categories: Prometheus
tags: Prometheus-Server
excerpt: rabbitmq exporter not up
mathjax: true
typora-root-url: ../
---

# rabbitmq_up

我们大概有4个dc，rabbitmq exporter的rabbitmq_up都是0，没有抓取到数据。看Log在curl rabbitmq管理端口的15672会报500错误

后来看了rabbitmq exporter文档才知道

### Extended RabbitMQ capabilities

Newer version of RabbitMQ can provide some features that reduce overhead imposed by scraping the data needed by this exporter. The following capabilities are currently supported in `RABBIT_CAPABILITIES` env var:

- `no_sort`: By default RabbitMQ management plugin sorts results using the default sort order of vhost/name. This sorting overhead can be avoided by passing empty sort argument (`?sort=`) to RabbitMQ starting from version 3.6.8. This option can be safely enabled on earlier 3.6.X versions, but it'll not give any performance improvements. And it's incompatible with 3.4.X and 3.5.X.
- `bert`: Since 3.6.9 (see https://github.com/rabbitmq/rabbitmq-management/pull/367) RabbitMQ supports BERT encoding as a JSON alternative. Given that BERT encoding is implemented in C inside the Erlang VM, it's way more effective than pure-Erlang JSON encoding. So this greatly reduces monitoring overhead when we have a lot of objects in RabbitMQ.

这个sort功能是默认enable的，但是呢，从rabbitmq 3.6.8之后才开始支持，我们的rabbitmq version是3.6.3

需要把`RABBIT_CAPABILITIES=nobert`配置上就可以了