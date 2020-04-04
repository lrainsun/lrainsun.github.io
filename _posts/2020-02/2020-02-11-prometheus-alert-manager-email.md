---
layout: post
title:  "alert manager配置概述"
date:   2020-02-11 23:00:00 +0800
categories: Prometheus
tags: Prometheus-AlertManager
excerpt: alert manager配置概述
mathjax: true
typora-root-url: ../
---

# 告警规则定义

Prometheus中的告警规则允许基于PromQL表达式定义告警触发条件，Prometheus后端对这些触发规则进行周期性计算，当满足触发条件后则会触发告警通知。

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

规则定义是yaml文件，比如

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

# Prometheus告警处理

Alertmanager可以对这些告警信息进行进一步的处理，比如消除重复的告警信息，对告警信息进行分组并且路由到正确的接受方。

## 分组

分组机制可以将分组后的一类告警信息合并成一个通知。比如

```yaml
route:
  group_by: ['alertname']
  group_wait: 1m
  group_interval: 1m
  repeat_interval: 24h
```

* 相同alertname的告警会被放到同一个通知里
* 这个通知会在等待group_wait时间后一起发送（如果在等待时间内当前group接收到了新的告警，这些告警将会合并为一个通知向receiver发送）
* 相同的Gourp之间发送告警通知的时间间隔是group_interval
* 如果警报已经成功发送通知，要等待repeat_interval时间再次发送告警

## 抑制

抑制是指当某一告警发出后，可以停止重复发送由此告警引发的其它告警的机制。通过Alertmanager的配置文件进行设置。

## 静默

静默提供了一个简单的机制可以快速根据标签对告警进行静默处理。如果接收到的告警符合静默的配置，Alertmanager则不会发送告警通知。需要在Alertmanager的Web页面上进行设置。

# 关联Prometheus与Alertmanager

在Prometheus的架构中被划分成两个独立的部分。

Prometheus负责产生告警，而Alertmanager负责告警产生后的后续处理。因此Alertmanager部署完成后，需要在Prometheus中设置Alertmanager相关的信息。

编辑Prometheus配置文件prometheus.yml，把两者关联起来

```yaml
alerting:
  alertmanagers:
  - scheme: http
    static_configs:
      - targets:
        - '10.225.12.133:9093'
```

# 告警接受者

在Alertmanager中路由负责对告警信息进行分组匹配，并将像告警接收器发送通知。告警接收也定义在配置文件里，可以通过route匹配不同的规则定义不同的接受者。

```yaml
receivers:
  - name: email
    email_configs:
    - to: 'lrain.sm@gmail.com'
      html: '{{ template "email.html" . }}'
      send_resolved: true
      
templates:
- '/etc/alertmanager/templates/email.tmpl'
```

还可以为告警的通知内容定义模板

```yaml
{{ define "email.html" }}
<table>
    <tr><td>报警名</td><td>开始时间</td></tr>
    {{ range $i, $alert := .Alerts }}
        <tr><td>{{ index $alert.Labels "alertname" }}</td><td>{{ $alert.StartsAt }}</td></tr>
    {{ end }}
</table>
{{ end }}
```



