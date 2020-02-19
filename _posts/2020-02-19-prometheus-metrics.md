---
layout: post
title:  "prometheus指标"
date:   2020-02-19 23:00:00 +0800
categories: Prometheus
tags: Prometheus-Server
excerpt: prometheus指标
mathjax: true
typora-root-url: ../
---

# prometheus指标

Prometheus的所有监控指标（Metric）被定义为指标&标签的组合

```shell
<metric_name>{<label_name>=<label_value>, ...}
```

## Metrics

指标名称用于说明指标的含义

指标名称必须由字面、数值下画线或者冒号组成，符合正则表达式`[a-zA-Z_:][a-zA-Z0-9_:]*`，exporter直接生成的指标不可以用“冒号”，冒号一般用于在prometheus server的recording rule

## Label

标签可体现指标的维度特征，用于过滤和聚合。

通过标签名（labelname）和标签值（label value）这种键值对的形式，形成多种维度。

以`__`开头的标签是在Prometheus系统内部使用的，比如每个metrics都会被打上`__name__=***`的标签。

# 指标分类

Prometheus指标分为Counter（计数器）、Gauge（仪表盘）、Histogram（直方图）和Summary（摘要）4类。

## Counter

* Counter是计数器类型，它的特点是只增不减，例如机器启动时间、HTTP访问量等。
* Counter 具有很好的不相关性，不会因为机器重启而置0。
* 在使用 Counter指标时，通常会结合 rate()方法获取该指标在某个时间段的变化率。

## Gauge

* Gauge是仪表盘，表征指标的实时变化情况，可增可减
* 大部分监控数据都是 Gauge 类型的

## Histogram

* Histogram是柱状图
* Histogram反映了某个区间内的样本个数，通过{le="上边界"}指定这个范围内的样本数。

![image-20200219223556068](/assets/images/image-20200219223556068.png)

* 对每个采样点进行统计（并不是一段时间的统计），打到各个bucket中
* 对每个采样点值累计和（sum）
* 对采样点的次数累计和（count）

![image-20200219224032527](/assets/images/image-20200219224032527.png)

### basename_bucket定义了各个桶中采样数据个数

采样数据值小于等于0.1（向下包含）的采样个数是2

采样数据值小于等于0.2的采样个数是2

……

采样数据值小于等于+Inf的采样个数是2

举的这个例子有点醉，没有很好体现。。所有的采样点都在一个篮子里了

### basename_sum定义了采样点的累计和

![image-20200219224453044](/assets/images/image-20200219224453044.png)

### basename_count定义了采样次数

![image-20200219225319567](/assets/images/image-20200219225319567.png)

所以很明显，一共就俩采样点，全都是小于0.1的，两个采样点加起来总和是0.00004215

### 百分位数（quantile, percentile）

百分位数是指小于某个特定数值的采样点达到一定的百分比

假设0.9-quantile的值为120，意思就是所有的采样值中，小于120的采样值的数量占总体采样值的90%。相应的，假设0.5-quantile的值为x，那么意思就是指小于x的采样值占总体的50%，所以0.5-quantile也指中值（median）

Histogram可以计算百分位数

xample: A histogram metric is called `http_request_duration_seconds`. To calculate the 90th percentile of request durations over the last 10m, use the following expression: [https://prometheus.io/docs/prometheus/latest/querying/functions/#histogram_quantile](https://prometheus.io/docs/prometheus/latest/querying/functions/#histogram_quantile)

```shell
histogram_quantile(0.9, rate(http_request_duration_seconds_bucket[10m]))
```

## Summary

* Summary 同 Histogram 一样，都属于高级指标，用于凸显数据的分布状况。
* 因为histogram在客户端就是简单的分桶和分桶计数，在prometheus服务端基于这么有限的数据做百分位估算，所以的确不是很准确，summary就是解决百分位准确的问题而来的
* **summary直接存储了 quantile 数据**，而不是根据统计区间计算出来的
* summary是采样点分位图统计

![image-20200219230957079](/assets/images/image-20200219230957079.png)

* 在客户端对一段时间的每个采样值进行统计，并形成分位图
* 统计所有采样值的总和
* 统计采样值的个数

### basename分位图

![image-20200219231307222](/assets/images/image-20200219231307222.png)

### basename_sum采样值和

![image-20200219231547149](/assets/images/image-20200219231547149.png)

### basename_count采样个数

![image-20200219231625126](/assets/images/image-20200219231625126.png)

