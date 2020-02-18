---
layout: post
title:  "prometheus alert rule被零除"
date:   2020-02-18 23:00:00 +0800
categories: Prometheus
tags: Prometheus-AlertManager
excerpt: prometheus alert rule被零除
mathjax: true
typora-root-url: ../
---

# prometheus alert rule被零除

这两天在体力劳动，整理alert rule，经常会有计算比例的情况，而分母为零也是很有可能出现的

当分母出现零值的时候，Metric的值就变成NaN，而我们通常不想要这样的值，想要诸如一个零值来代替

一开始我们的解决方法是 `分子/（分母+1）`，这样就可以保证分母永远不为0

我查了一下，可以有其他解决方法：

`分子/分母 > 0 or 分组 > bool 0 `

1. 当分母不为0才执行前面的除法运算
2. 分母为0的时候永远返回0

```yaml
  - expr: interface:error_transmit_packets:1m/((interface:dropped_transmit_packets:1m + interface:error_transmit_packets:1m + interface:transmit_packets:1m) > 0) or (interface:dropped_transmit_packets:1m + interface:error_transmit_packets:1m + interface:transmit_packets:1m) > bool 0
    record: 'interface:error_transmit_packets:ratio_rate1m'
```

