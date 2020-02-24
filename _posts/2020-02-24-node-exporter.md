---
layout: post
title:  "node exporter代码"
date:   2020-02-24 23:00:00 +0800
categories: Prometheus
tags: Prometheus-Exporter
excerpt: node exporter代码
mathjax: true
typora-root-url: ../
---

# node exporter

入口在node_exporter.go，node_exporter监听在9100端口

```go
func main() {
	var (
		listenAddress = kingpin.Flag(
			"web.listen-address",
			"Address on which to expose metrics and web interface.",
		).Default(":9100").String()
		metricsPath = kingpin.Flag(
			"web.telemetry-path",
			"Path under which to expose metrics.",
		).Default("/metrics").String()
		disableExporterMetrics = kingpin.Flag(
			"web.disable-exporter-metrics",
			"Exclude metrics about the exporter itself (promhttp_*, process_*, go_*).",
		).Bool()
		maxRequests = kingpin.Flag(
			"web.max-requests",
			"Maximum number of parallel scrape requests. Use 0 to disable.",
		).Default("40").Int()
		disableDefaultCollectors = kingpin.Flag(
			"collector.disable-defaults",
			"Set all collectors to disabled by default.",
		).Default("false").Bool()
		configFile = kingpin.Flag(
			"web.config",
			"[EXPERIMENTAL] Path to config yaml file that can enable TLS or authentication.",
		).Default("").String()
	)
  ......
  http.Handle(*metricsPath, newHandler(!*disableExporterMetrics, *maxRequests, logger))
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`<html>
			<head><title>Node Exporter</title></head>
			<body>
			<h1>Node Exporter</h1>
			<p><a href="` + *metricsPath + `">Metrics</a></p>
			</body>
			</html>`))
	})
  level.Info(logger).Log("msg", "Listening on", "address", *listenAddress)
	server := &http.Server{Addr: *listenAddress}
	if err := https.Listen(server, *configFile); err != nil {
		level.Error(logger).Log("err", err)
		os.Exit(1)
	}
}
```

```go
// ServeHTTP implements http.Handler.
func (h *handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	filters := r.URL.Query()["collect[]"]
	level.Debug(h.logger).Log("msg", "collect query:", "filters", filters)

	if len(filters) == 0 {
		// No filters, use the prepared unfiltered handler.
		h.unfilteredHandler.ServeHTTP(w, r)
		return
	}
	// To serve filtered metrics, we create a filtering handler on the fly.
	filteredHandler, err := h.innerHandler(filters...)
	if err != nil {
		level.Warn(h.logger).Log("msg", "Couldn't create filtered metrics handler:", "err", err)
		w.WriteHeader(http.StatusBadRequest)
		w.Write([]byte(fmt.Sprintf("Couldn't create filtered metrics handler: %s", err)))
		return
	}
	filteredHandler.ServeHTTP(w, r)
}
```

innerHandler里创建了一个新的NodeCollector

```go
func (h *handler) innerHandler(filters ...string) (http.Handler, error) {
	nc, err := collector.NewNodeCollector(h.logger, filters...)
	if err != nil {
		return nil, fmt.Errorf("couldn't create collector: %s", err)
	}
}
```

NodeCollector在定义在collector/collector.go

```go
// NodeCollector implements the prometheus.Collector interface.
type NodeCollector struct {
	Collectors map[string]Collector
	logger     log.Logger
}
```

可以看到NodeCollector是包含了各种Collector的集合，每个Collector在启动时都会将自身注册到这个Collectors中

例如collector/cpu_linux.go中的 NewCPUCollector在init时，会把自己注册进去

```go
func init() {
	registerCollector("cpu", defaultEnabled, NewCPUCollector)
}
```

NodeCollector 实现了 Prometheus.Collector 接口，会被 Prometheus 的客户端库调用，在 Collect中遍历每个Collectors采集器，并执行采集动作

```go
// Collect implements the prometheus.Collector interface.
func (n NodeCollector) Collect(ch chan<- prometheus.Metric) {
	wg := sync.WaitGroup{}
	wg.Add(len(n.Collectors))
	for name, c := range n.Collectors {
		go func(name string, c Collector) {
			execute(name, c, ch, n.logger)
			wg.Done()
		}(name, c)
	}
	wg.Wait()
}
```

execute会调用Update()完成采集，每个collector的update()都会被调用

```go
func execute(name string, c Collector, ch chan<- prometheus.Metric, logger log.Logger) {
	begin := time.Now()
	err := c.Update(ch)
	duration := time.Since(begin)
	var success float64
}
```

比如刚刚说的CPUCollector，prometheus.Metric是一个只读channel，获取到的cpu信息会发送到这个channel中。update()方法会周期性地被调用。

```go
func (c *cpuCollector) Update(ch chan<- prometheus.Metric) error {
	if *enableCPUInfo {
		if err := c.updateInfo(ch); err != nil {
			return err
		}
	}
	if err := c.updateStat(ch); err != nil {
		return err
	}
	if err := c.updateThermalThrottle(ch); err != nil {
		return err
	}
	return nil
}
```

