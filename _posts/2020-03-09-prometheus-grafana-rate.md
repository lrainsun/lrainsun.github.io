---
layout: post
title:  "rate取不到数据"
date:   2020-03-09 23:00:00 +0800
categories: Prometheus
tags: Prometheus-Server
excerpt: rate取不到数据
mathjax: true
typora-root-url: ../
---

# rate取不到数据

今天发现我们grafana上有个CPU Utilisation取不到数据

![image-20200309183452568](/assets/images/image-20200309183452568.png)

先看下rate的定义

rate是用来计算某个指标在一定时间间隔内的变化速率的。另外还有一个指标irate，也是同样的功能。只是他们计算方式有所不同：

* rate：取指定时间范围内所有数据点，算出一组速率，然后取平均值作为结果 —— 适合变化缓慢的计数器
* irate：取的是在指定时间范围内的最近两个数据点来算速率 —— 适合快速变化的计数器

所以，基于如上定义，`The rate and irate functions need at least two scrapes to return a result.` rate跟irate需要至少两个值来计算

在我们的配置里

```yaml
- job_name: federate
  honor_labels: true
  honor_timestamps: true
  params:
    match[]:
    - '{__name__=~".+"}'
  scrape_interval: 1m
  scrape_timeout: 1m
  metrics_path: /prometheus/federate
  scheme: https
  static_configs:
```

scrape_interval是1m，所以在1m我们只能取到一个采样值，这样就没办法完成rate的计算

调整 `1 - avg(rate(node_cpu_seconds_total{mode="idle"}[1m]))`到`1 - avg(rate(node_cpu_seconds_total{mode="idle"}[2m]))`，就可以成功获取到了

