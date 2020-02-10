---
layout: post
title:  "Federation prometheus拉取数据match配置"
date:   2020-02-10 23:00:00 +0800
categories: Prometheus
tags: Prometheus-Server
excerpt: Federation prometheus拉取数据match配置
mathjax: true
typora-root-url: ../
---

# match配置

我们在每个DC都有一个local的prometheus用来拉取本地环境的metrics数据，想要配置一个federation prometheus，把所有DC的数据都拉过来

```yaml
- job_name: federate
  scrape_interval: 15s
  honor_labels: true
  scheme: https
  metrics_path: '/federate'
  params:
    'match[]':
      - '{__name__=~".+"}'
  static_configs:
    - targets:
      - 'ci22am-prometheus.webex.com:443'
      - 'ci32am-prometheus.webex.com:443'
      - 'ci22ln-prometheus.webex.com:443'
      - 'ci32ln-prometheus.webex.com:443'
      - 'ci22jp-prometheus.webex.com:443'
      - 'ci32jp-prometheus.webex.com:443'
      - 'ci22sg-prometheus.webex.com:443'
      - 'ci32sg-prometheus.webex.com:443'
      - 'ci32sj-prometheus.webex.com:443'
      - 'ci32ta-prometheus.webex.com:443'
```

{__name__=~".+"} 表示拉取所有的metrics

但是我们有个需求想要过滤所有name类似这样“nova_hypervisor_memory:usage” 带冒号的metrics

可以这样：

* 我们不要`__name__!~".*:.*"`的数据，可是只写这一个条件会报错
  Error executing query:  invalid parameter 'query': parse error at char 19: vector selector must  contain at least one non-empty matcher

* 所以可以加上匹配所有的job`job=~".+"`

如果这样写，两者是or的关系，还是不行

```yaml
  params:
    'match[]':
      - '{__name__!~".*:.*}'
      - '{`job=~".+"}'
```

所以要写成这样才满足我们的要求

```yaml
  params:
    'match[]':
      - '{__name__!~".*:.*",job=~".+"}'
```

