---
layout: post
title:  "openstack exporter启动失败3"
date:   2021-02-07 23:00:00 +0800
categories: Prometheus
tags: Prometheus-Exporter
excerpt: openstack exporter启动失败3
mathjax: true
typora-root-url: ../
---

# openstack exporter启动失败3

哈哈，还有后续，后来想了下，在sdk里面直接指定版本属实有点不好

所以后来还是决定回Openstack exporter里改

```go
func NewNovaExporter(config *ExporterConfig) (*NovaExporter, error) {
»·exporter := NovaExporter{
»·»·BaseOpenStackExporter{
»·»·»·Name:           "nova",
»·»·»·ExporterConfig: *config,
»·»·},
»·}
»·for _, metric := range defaultNovaMetrics {
»·»·if !exporter.isSlowMetric(&metric) {
»·»·»·exporter.AddMetric(metric.Name, metric.Fn, metric.Labels, nil)
»·»·}
»·}
»·microversion, err := apiversions.Get(config.Client, "v2.1").Extract()
»·if err == nil {
»·»·exporter.Client.Microversion = microversion.Version
»·}

»·return &exporter, nil
}
```

如果可以获取到主版本是2.1的microversion，那么就用那个microversion，根据server端的支持能力来请求