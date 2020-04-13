---
layout: post
title:  "rabbitmq exporter超时"
date:   2020-04-13 23:00:00 +0800
categories: Prometheus
tags: Prometheus-Server
excerpt: rabbitmq exporter超时
mathjax: true
typora-root-url: ../
---

# rabbitmq exporter超时

在我们目前最大的数据中心rabbitmq exporter经常取不到数据，查看打印：

```shell
time="2020-04-13T05:59:55Z" level=info msg="Active Configuration" CAFILE=ca.pem EXCLUDE_METRICS="[]" INCLUDE_QUEUES=".*" INCLUDE_VHOST=".*" MAX_QUEUES=0 OUTPUT_FORMAT=TTY PUBLISH_ADDR=10.252.126.79 PUBLISH_PORT=9419 RABBIT_CAPABILITIES="bert,no_sort" RABBIT_EXPORTERS="[exchange node overview queue]" RABBIT_TIMEOUT=30 RABBIT_URL="http://10.252.126.79:15672" RABBIT_USER=rabbituser SKIPVERIFY=false SKIP_QUEUES="^$" SKIP_VHOST="^$"
time="2020-04-13T06:03:10Z" level=error msg="Error while retrieving data from rabbitHost" error="Get http://10.252.126.79:15672/api/queues?sort=: net/http: request canceled (Client.Timeout exceeded while awaiting headers)" host="http://10.252.126.79:15672" statusCode=0
time="2020-04-13T06:03:10Z" level=warning msg="retrieving queue failed" error="Error while retrieving data from rabbitHost"
```

原因是数据量太大，超时了

当前设置的timeout是30 `RABBIT_TIMEOUT=30 `

可以设置长一些，另外还有两个参数可以调节

```shell
no_sort: By default RabbitMQ management plugin sorts results using
the default sort order of vhost/name. This sorting overhead can be
avoided by passing empty sort argument (?sort=) to RabbitMQ
starting from version 3.6.8. This option can be safely enabled on
earlier 3.6.X versions, but it'll not give any performance
improvements. And it's incompatible with 3.4.X and 3.5.X.

bert: Since 3.6.9 (see
rabbitmq/rabbitmq-management#367) RabbitMQ
supports BERT encoding as a JSON alternative. Given that BERT
encoding is implemented in C inside the Erlang VM, it's way more
effective than pure-Erlang JSON encoding. So this greatly reduces
monitoring overhead when we have a lot of objects in RabbitMQ.
```

一个是不排序，一个是用bert encoding代替json，效率可以有所提高

```shell
RABBIT_CAPABILITIES=bert,no_sort
RABBIT_TIMEOUT=60
```

修改后

```shell
time="2020-04-13T06:16:39Z" level=info msg="Metrics updated" duration=40.175754244s
time="2020-04-13T06:20:01Z" level=info msg="Metrics updated" duration=40.339136242s
time="2020-04-13T06:21:07Z" level=info msg="Metrics updated" duration=11.898321558s
time="2020-04-13T06:22:01Z" level=info msg="Metrics updated" duration=12.048741477s
time="2020-04-13T06:23:11Z" level=info msg="Metrics updated" duration=40.451194021s
```

