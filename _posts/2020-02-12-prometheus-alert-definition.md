---
layout: post
title:  "alert manager规则定义"
date:   2020-02-12 23:00:00 +0800
categories: Prometheus
tags: Prometheus-AlertManager
excerpt: alert manager规则定义
mathjax: true
typora-root-url: ../
---

# 告警规则定义

Prometheus中的告警规则允许你基于PromQL表达式定义告警触发条件，Prometheus后端对这些触发规则进行周期性计算，当满足触发条件后则会触发告警通知。默认情况下，用户可以通过Prometheus的Web界面查看这些告警规则以及告警的触发状态。当Promthues与Alertmanager关联之后，可以将告警发送到外部服务如Alertmanager中并通过Alertmanager可以对这些告警进行进一步的处理。

## 告警触发规则计算

```yaml
global:
  evaluation_interval: 1m
```

evaluation_interval等于1m，就说明每1分钟会去计算告警的触发条件。所谓告警的触发条件，指的是在rule文件里定义的 `- alert:`字样的条件

## 告警规则文件

可以在Prometheus全局配置文件中通过**rule_files**指定一组告警规则文件的访问路径。Prometheus启动后会自动扫描这些路径下规则文件中定义的内容，并且根据这些规则计算是否向外部发送通知。

```yaml
rule_files:
- /etc/prometheus/record.rules
- /etc/prometheus/haproxy_rule.rules
- /etc/prometheus/memcached_rule.rules
- /etc/prometheus/mysql_rule.rules
- /etc/prometheus/rabbitmq_rule.rules
- /etc/prometheus/ceph_rule.rules
- /etc/prometheus/slo_30day.rules
- /etc/prometheus/node_rule.rules
- /etc/prometheus/libvirt_rule.rules
- /etc/prometheus/slo_1min.rules
```

## 告警规则定义

定义在rule file里的有两者，一种是recording rules，一种是alerting rules。

### rule group

这两种rule可以存在于同一个rule group下，对于一个rule group，可以定义group name，group interval，这意味着同一个group下的rules都根据同一个interval来计算数据（如果没定义，就用刚刚上面讲到的global配置里的默认值）。

```yaml
groups:
  - name: openstack.rules
    rules:
    - expr: sum (nova_servers) by (cloud, flavor, job)
      record: 'flavor:nova_servers:sum'
    interval: 1m
```

### recording rules

recording rules可以对一些原始数据值进行预计算，然后计算得出的结果会被存成新的time series的记录。

```yaml
groups:
  - name: openstack.rules
    rules:
    - expr: sum (nova_servers) by (cloud, flavor, job)
      record: 'flavor:nova_servers:sum'
```

recording rules可以定义：

* record：time series metric的名字
* expr：PromQL表达式
* labels：可以给计算结果添加或者覆盖原有lables

### alerting rules

```yaml
groups:
- name: haproxy
  rules:
  - alert: HaproxyProcessDown
    expr: haproxy_up{job="haproxy"} == 0
    for: 1m
    labels:
      severity: average
    annotations:
      description: "Haproxy process is down on {{ $labels.instance }}"
```

alerting rules可以定义：

* alert：alert的名字
* expr：PromQL表达式，用来定义是否会变成pending/firing告警
* for：一个alerting rule满足条件后，经过for定义的时间后，才会变成firing，在变成firing之前是pending状态
* labels：可以给alert添加或者覆盖原有lables

* annotations：alert的注释，可以有summary和description

