---
layout: post
title:  "openstack exporter启动失败"
date:   2021-02-04 23:00:00 +0800
categories: Prometheus
tags: Prometheus-Exporter
excerpt: openstack exporter启动失败
mathjax: true
typora-root-url: ../
---

# openstack exporter启动失败

我们在启用prometheus openstack exporter的时候，总是遇到这样的问题：

```shell
[root@edge-2-ocp-dev-sjc02-d ~]# curl http://edge-2.ocp-dev-sjc02-d.cloud.prv.webex.com:9198/metrics
An error has occurred while serving metrics:

collected metric "openstack_nova_agent_state" { label:<name:"adminState" value:"enabled" > label:<name:"disabledReason" value:"" > label:<name:"hostname" value:"ctrl-1.ocp-dev-sjc02-d.cloud.prv.webex.com" > label:<name:"id" value:"2" > label:<name:"service" value:"nova-conductor" > label:<name:"zone" value:"internal" > counter:<value:1 > } was collected before with the same name and label values
```

换了好多image也不管用，之前19年有过这个[bug](https://github.com/openstack-exporter/openstack-exporter/issues/37)，但是看起来，我们已经是新代码了，label里已经有ID，那为啥还有问题呢？

又仔细读了一下错误日志，跑到Openstack环境看了一下才知道，原来

![image-20210204225746822](/../assets/images/image-20210204225746822.png)

启用了cell之后，nova conductor有两个，一个是nova conductor，一个是nova super conductor，但是呢他们的名字又都叫nova-conductor

```
| 90a34d86-d651-4fe0-ba8a-2ef92aec275e | nova-conductor | ctrl-1.ocp-dev-sjc02-d.cloud.prv.webex.com | internal | enabled | up | 2021-02-04T09:14:49.000000 | - | False |
| 9e47adc8-8302-4d21-a26f-cad1b70b4fc1 | nova-conductor | ctrl-1.ocp-dev-sjc02-d.cloud.prv.webex.com | internal | enabled | up | 2021-02-04T09:14:50.000000 | - | False |
```

一个在cell0数据库里边，一个在cell1库里

而且就这么巧，ctrl-1上的nova-conductor跟nova-super-conductor连ID都是一样的

```shell
| 2021-02-03 03:04:23 | 2021-02-04 09:24:09 | NULL | 2 | ctrl-1.ocp-dev-sjc02-d.cloud.prv.webex.com | nova-conductor | conductor | 10916 | 0 | 0 | NULL | 2021-02
| 2021-02-03 03:05:23 | 2021-02-04 09:24:40 | NULL | 2 | ctrl-1.ocp-dev-sjc02-d.cloud.prv.webex.com | nova-conductor | conductor | 10914 | 0 | 0 | NULL | 2021-02-04 0
```

这个其实是很难避免的，直接用的数据库的seq id，不同表容易出现重复

```shell
MariaDB [nova_cell0]> select * from services;
+---------------------+---------------------+------------+-----+--------------------------------------------+--------------------+-----------+--------------+----------+---------+-----------------+---------------------+-------------+---------+--------------------------------------+
| created_at          | updated_at          | deleted_at | id  | host                                       | binary             | topic     | report_count | disabled | deleted | disabled_reason | last_seen_up        | forced_down | version | uuid                                 |
+---------------------+---------------------+------------+-----+--------------------------------------------+--------------------+-----------+--------------+----------+---------+-----------------+---------------------+-------------+---------+--------------------------------------+
| 2021-02-03 03:04:23 | 2021-02-04 14:06:33 | NULL       |   2 | ctrl-1.ocp-dev-sjc02-d.cloud.prv.webex.com | nova-conductor     | conductor |        12610 |        0 |       0 | NULL            | 2021-02-04 14:06:33 |           0 |      51 | 90a34d86-d651-4fe0-ba8a-2ef92aec275e |
| 2021-02-03 03:04:28 | 2021-02-04 14:06:27 | NULL       |  11 | ctrl-3.ocp-dev-sjc02-d.cloud.prv.webex.com | nova-conductor     | conductor |        12607 |        0 |       0 | NULL            | 2021-02-04 14:06:27 |           0 |      51 | 674dd729-eb6f-46ce-aeef-9fc5fc26595c |
| 2021-02-03 03:04:49 | 2021-02-04 14:06:37 | NULL       |  26 | ctrl-2.ocp-dev-sjc02-d.cloud.prv.webex.com | nova-conductor     | conductor |        12605 |        0 |       0 | NULL            | 2021-02-04 14:06:37 |           0 |      51 | 209e3b79-910d-4387-9d9f-0fade1b4f66a |
```

感觉上是可以用uuid作为label更好

那我们就需要修改[openstack exporter](https://github.com/openstack-exporter/openstack-exporter)的代码，其实代码修改不难，在exporters/nova.go，agent_state我尝试加了service的uuid，但是，改好之后`go build`，提示如下错误：


```go
exporters/nova.go:121:52: service.UUID undefined (type "github.com/gophercloud/gophercloud/openstack/compute/v2/extensions/services".Service has no field or method UUID)
```

在go的依赖库里面`https://github.com/gophercloud/gophercloud/blob/master/openstack/compute/v2/extensions/services/results.go`

```go
type Service struct {
	// The binary name of the service.
	Binary string `json:"binary"`

	// The reason for disabling a service.
	DisabledReason string `json:"disabled_reason"`

	// Whether or not service was forced down manually.
	ForcedDown bool `json:"forced_down"`

	// The name of the host.
	Host string `json:"host"`

	// The id of the service.
	ID string `json:"-"`

	// The state of the service. One of up or down.
	State string `json:"state"`

	// The status of the service. One of enabled or disabled.
	Status string `json:"status"`

	// The date and time when the resource was updated.
	UpdatedAt time.Time `json:"-"`

	// The availability zone name.
	Zone string `json:"zone"`
}
```

service并没有提供uuid。。那么我就挑个UpdatedAt来做区分吧。。

最终修改nova.go代码如下：

```go
{Name: "agent_state", Labels: []string{"id", "hostname", "service", "adminState", "zone", "disabledReason", "UpdatedAt"}, Fn: ListNovaAgentState},

                ch <- prometheus.MustNewConstMetric(exporter.Metrics["agent_state"].Metric,
                        prometheus.CounterValue, float64(state), service.ID, service.Host, service.Binary, service.Status, service.Zone, service.DisabledReason, service.UpdatedAt.String())
```

然后go build，然后把编译后的二进制文件拷贝到docker/prometheus/prometheus-openstack-exporter/，并修改docker/prometheus/prometheus-openstack-exporter/Dockerfile.j2

```makefile
RUN mkdir -p /opt/openstack-exporter
COPY openstack-exporter /opt/openstack-exporter/openstack-exporter
```