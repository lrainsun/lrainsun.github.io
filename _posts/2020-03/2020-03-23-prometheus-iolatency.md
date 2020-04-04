---
layout: post
title:  "disk io latency"
date:   2020-03-23 23:00:00 +0800
categories: Prometheus
tags: Prometheus-Server
excerpt: disk io latency
mathjax: true
typora-root-url: ../
---

# Latency

**Latency** 又指responding time，是指完成一个IO请求所有需要的时间。通常是以milliseconds来衡量的。

除此之外：

**IOPS**：Input/Output Per Second， 即每秒系统能能处理的I/O请求数量。

**TPUT**：吞吐量。

在prometheus node-exporter里对于read/write有两组Metrics可以用来计算io latency

write

- `node_disk_write_time_seconds_total`（This is the total number of seconds spent by all writes.）
- `node_disk_writes_completed_total`（The total number of writes completed successfully.）

read

- `node_disk_read_time_seconds_total`（The total number of seconds spent by all reads.）
- `node_disk_reads_completed_total`（The total number of reads completed successfully.）

这里我们可以用到一个Prometheus内置函数rate()

`rate(v range-vector)` 函数可以直接计算区间向量 v 在时间窗口内平均增长速率，它会在单调性发生变化时(如由于采样目标重启引起的计数器复位)自动中断。该函数的返回结果**不带有度量指标**，只有标签列表。

`rate(node_disk_read_time_seconds_total[5m])` —— 把每5分钟的采样数据的平均值，也就是平均每秒花在读上的时间

`rate(node_disk_reads_completed_total[5m])` —— 5分钟完成了多少IO

`rate(node_disk_read_time_seconds_total[5m]) / rate(node_disk_reads_completed_total[5m])` —— 完成一个IO花费了多少时间，也就是latency了