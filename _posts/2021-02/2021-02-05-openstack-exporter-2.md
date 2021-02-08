---
layout: post
title:  "openstack exporter启动失败2"
date:   2021-02-05 23:00:00 +0800
categories: Prometheus
tags: Prometheus-Exporter
excerpt: openstack exporter启动失败2
mathjax: true
typora-root-url: ../
---

# openstack exporter启动失败2

没想到昨天的问题还有后续，早上发现openstack exporter又挂了，还是一样的错误

```shell
[root@edge-2 ~]# curl 192.168.0.3:9198/metrics
An error has occurred while serving metrics:

collected metric "openstack_nova_agent_state" { label:<name:"UpdatedAt" value:"2021-02-05 08:45:19 +0000 UTC" > label:<name:"adminState" value:"enabled" > label:<name:"disabledReason" value:"" > label:<name:"hostname" value:"ctrl-1.ocp-dev-sjc02-d.cloud.prv.webex.com" > label:<name:"id" value:"2" > label:<name:"service" value:"nova-conductor" > label:<name:"zone" value:"internal" > counter:<value:1 > } was collected before with the same name and label values
```

这个updateAt时间也会一直更新的，所以有可能变成一样了，那这个问题就麻烦了

所以想查一下uuid该怎么获取到，就看了[nova restful api](https://docs.openstack.org/api-ref/compute/?expanded=list-compute-services-detail#list-compute-services)文档，看到这个

![image-20210205165741087](/../assets/images/image-20210205165741087.png)

原来2.53版本之后，ID返回的就是UUID了

```shell
MINSU-M-M1RW:openstack-exporter-rain-2 minsu$ nova --os-compute-api-version 2.52 service-list
+-----+----------------+--------------------------------------------+----------+---------+-------+----------------------------+-----------------+-------------+
| Id  | Binary         | Host                                       | Zone     | Status  | State | Updated_at                 | Disabled Reason | Forced down |
+-----+----------------+--------------------------------------------+----------+---------+-------+----------------------------+-----------------+-------------+
| 2   | nova-conductor | ctrl-1.ocp-dev-sjc02-d.cloud.prv.webex.com | internal | enabled | up    | 2021-02-05T06:21:57.000000 | -               | False       |
| 11  | nova-conductor | ctrl-3.ocp-dev-sjc02-d.cloud.prv.webex.com | internal | enabled | up    | 2021-02-05T06:21:50.000000 | -               | False       |
| 26  | nova-conductor | ctrl-2.ocp-dev-sjc02-d.cloud.prv.webex.com | internal | enabled | up    | 2021-02-05T06:21:57.000000 | -               | False       |
| 41  | nova-scheduler | ctrl-3.ocp-dev-sjc02-d.cloud.prv.webex.com | internal | enabled | up    | 2021-02-05T06:21:49.000000 | -               | False       |
| 68  | nova-scheduler | ctrl-1.ocp-dev-sjc02-d.cloud.prv.webex.com | internal | enabled | up    | 2021-02-05T06:21:52.000000 | -               | False       |
| 122 | nova-scheduler | ctrl-2.ocp-dev-sjc02-d.cloud.prv.webex.com | internal | enabled | up    | 2021-02-05T06:21:54.000000 | -               | False       |
| 2   | nova-conductor | ctrl-1.ocp-dev-sjc02-d.cloud.prv.webex.com | internal | enabled | up    | 2021-02-05T06:21:57.000000 | -               | False       |
| 17  | nova-conductor | ctrl-3.ocp-dev-sjc02-d.cloud.prv.webex.com | internal | enabled | up    | 2021-02-05T06:21:52.000000 | -               | False       |
| 32  | nova-conductor | ctrl-2.ocp-dev-sjc02-d.cloud.prv.webex.com | internal | enabled | up    | 2021-02-05T06:21:48.000000 | -               | False       |
| 47  | nova-compute   | ctrl-3-ironic                              | nova     | enabled | up    | 2021-02-05T06:21:55.000000 | -               | False       |
| 50  | nova-compute   | ctrl-2-ironic                              | nova     | enabled | up    | 2021-02-05T06:21:54.000000 | -               | False       |
| 53  | nova-compute   | ctrl-1-ironic                              | nova     | enabled | up    | 2021-02-05T06:21:52.000000 | -               | False       |
| 56  | nova-compute   | cmp-1.ocp-dev-sjc02-d.cloud.prv.webex.com  | shared   | enabled | up    | 2021-02-05T06:21:51.000000 | -               | False       |
+-----+----------------+--------------------------------------------+----------+---------+-------+----------------------------+-----------------+-------------+
MINSU-M-M1RW:openstack-exporter-rain-2 minsu$ nova --os-compute-api-version 2.53 service-list
+--------------------------------------+----------------+--------------------------------------------+----------+---------+-------+----------------------------+-----------------+-------------+
| Id                                   | Binary         | Host                                       | Zone     | Status  | State | Updated_at                 | Disabled Reason | Forced down |
+--------------------------------------+----------------+--------------------------------------------+----------+---------+-------+----------------------------+-----------------+-------------+
| 90a34d86-d651-4fe0-ba8a-2ef92aec275e | nova-conductor | ctrl-1.ocp-dev-sjc02-d.cloud.prv.webex.com | internal | enabled | up    | 2021-02-05T06:21:57.000000 | -               | False       |
| 674dd729-eb6f-46ce-aeef-9fc5fc26595c | nova-conductor | ctrl-3.ocp-dev-sjc02-d.cloud.prv.webex.com | internal | enabled | up    | 2021-02-05T06:22:00.000000 | -               | False       |
| 209e3b79-910d-4387-9d9f-0fade1b4f66a | nova-conductor | ctrl-2.ocp-dev-sjc02-d.cloud.prv.webex.com | internal | enabled | up    | 2021-02-05T06:22:07.000000 | -               | False       |
| a6154c2e-de24-4b94-83bf-3f9af663ec80 | nova-scheduler | ctrl-3.ocp-dev-sjc02-d.cloud.prv.webex.com | internal | enabled | up    | 2021-02-05T06:21:59.000000 | -               | False       |
| f62ad8cb-89fe-4b57-a490-f4189cb85106 | nova-scheduler | ctrl-1.ocp-dev-sjc02-d.cloud.prv.webex.com | internal | enabled | up    | 2021-02-05T06:22:02.000000 | -               | False       |
| 4dd7ddd7-4a89-41a0-a92b-5060c5bd7467 | nova-scheduler | ctrl-2.ocp-dev-sjc02-d.cloud.prv.webex.com | internal | enabled | up    | 2021-02-05T06:22:04.000000 | -               | False       |
| 9e47adc8-8302-4d21-a26f-cad1b70b4fc1 | nova-conductor | ctrl-1.ocp-dev-sjc02-d.cloud.prv.webex.com | internal | enabled | up    | 2021-02-05T06:21:57.000000 | -               | False       |
| 94dd4484-b730-4f93-b593-273eda554bc7 | nova-conductor | ctrl-3.ocp-dev-sjc02-d.cloud.prv.webex.com | internal | enabled | up    | 2021-02-05T06:22:02.000000 | -               | False       |
| 71bf5e32-c98e-40cb-8e7e-2740325722bd | nova-conductor | ctrl-2.ocp-dev-sjc02-d.cloud.prv.webex.com | internal | enabled | up    | 2021-02-05T06:21:58.000000 | -               | False       |
| 183ec633-23f5-4e67-9bec-420b9560f5b8 | nova-compute   | ctrl-3-ironic                              | nova     | enabled | up    | 2021-02-05T06:22:05.000000 | -               | False       |
| 51a00d8a-a7fe-416c-9857-095c168045c3 | nova-compute   | ctrl-2-ironic                              | nova     | enabled | up    | 2021-02-05T06:22:04.000000 | -               | False       |
| c0231aed-d4af-495c-a015-3cb9e9632bb3 | nova-compute   | ctrl-1-ironic                              | nova     | enabled | up    | 2021-02-05T06:22:02.000000 | -               | False       |
| bc7b1300-7832-4c61-a1a9-778cbf28db9d | nova-compute   | cmp-1.ocp-dev-sjc02-d.cloud.prv.webex.com  | shared   | enabled | up    | 2021-02-05T06:22:01.000000 | -               | False       |
+--------------------------------------+----------------+--------------------------------------------+----------+---------+-------+----------------------------+-----------------+-------------+
```

所以问题就是我们需要让get service的时候去指定api 版本`--header 'X-OpenStack-Nova-API-Version: 2.53'`

而这个事情并不是在Openstack exporter里直接做的，而是调用了go库gophercloud，看了代码

```go
## Client Configuration

You can set a specific microversion on a Service Client by doing the following:

​```go
client, err := openstack.NewComputeV2(providerClient, nil)
client.Microversion = "2.52"
```

所以修改了代码`gophercloud/openstack/compute/v2/extensions/services/requests.go`

```go
// List makes a request against the API to list services.
func List(client *gophercloud.ServiceClient, opts ListOptsBuilder) pagination.Pager {
        client.Microversion = "2.53"
        url := listURL(client)
        if opts != nil {
                query, err := opts.ToServicesListQuery()
                if err != nil {
                        return pagination.Pager{Err: err}
                }
                url += query
        }

        return pagination.NewPager(client, url, func(r pagination.PageResult) pagination.Page {
                return ServicePage{pagination.SinglePageBase(r)}
        })
}
```

改完之后要怎么打包呢？

看`go.mod`

```go
module github.com/openstack-exporter/openstack-exporter

go 1.13

require (
	github.com/gophercloud/gophercloud v0.15.0
	github.com/gophercloud/utils v0.0.0-20200918191848-da0e919a012a
	github.com/hashicorp/go-uuid v1.0.1
	github.com/jarcoal/httpmock v1.0.4
	github.com/prometheus/client_golang v1.2.1
	github.com/prometheus/client_model v0.0.0-20191202183732-d1d2010b5bee // indirect
	github.com/prometheus/common v0.7.0
	github.com/stretchr/testify v1.4.0
	gopkg.in/alecthomas/kingpin.v2 v2.2.6
)
```

这里会默认去download依赖包

```
echo $GOPATH
/usr/local/Cellar/go/1.13.3
如果工程中存在go.mod文件，编译时是从GOPATH/pkg/mod下查找依赖。

MINSU-M-M1RW:openstack-exporter minsu$ ls -al /usr/local/Cellar/go/1.13.3/pkg/mod/github.com/gophercloud/
total 0
drwxr-xr-x   5 minsu  admin   160 Feb  5 15:41 .
drwxr-xr-x  41 minsu  admin  1312 Feb  4 21:56 ..
dr-x------  28 minsu  admin   896 Feb  4 21:56 gophercloud@v0.15.0
```

但是因为我们修改了代码，所以想用本地的依赖，可以加一个replace

```go
module github.com/openstack-exporter/openstack-exporter

go 1.13

require (
	github.com/gophercloud/gophercloud v0.15.0
	github.com/gophercloud/utils v0.0.0-20200918191848-da0e919a012a
	github.com/hashicorp/go-uuid v1.0.1
	github.com/jarcoal/httpmock v1.0.4
	github.com/prometheus/client_golang v1.2.1
	github.com/prometheus/client_model v0.0.0-20191202183732-d1d2010b5bee // indirect
	github.com/prometheus/common v0.7.0
	github.com/stretchr/testify v1.4.0
	gopkg.in/alecthomas/kingpin.v2 v2.2.6
)

replace github.com/gophercloud/gophercloud v0.15.0 => /root/openstack-exporter/gophercloud
```

然后就可以一样go build，然后跟昨天一样把生成的二进制拿到kolla那边去build image