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

那我们就需要修改[openstack exporter](https://github.com/openstack-exporter/openstack-exporter)的代码，其实代码修改不难，在exporters/nova.go，agent_state我尝试加了service的uuid，但是，改好之后`go build`，提示如下错误，在go的依赖库里面`https://github.com/gophercloud/gophercloud/blob/master/openstack/compute/v2/extensions/services/results.go`

service并没有提供uuid。。那么我就挑个UpdatedAt来做区分吧。。

最终修改nova.go代码增加UpdatedAt

然后go build，然后把编译后的二进制文件拷贝到docker/prometheus/prometheus-openstack-exporter/，并修改docker/prometheus/prometheus-openstack-exporter/Dockerfile.j2

```makefile
FROM {{ namespace }}/{{ infra_image_prefix }}prometheus-base:{{ tag }}
{% block labels %}
LABEL maintainer="{{ maintainer }}" name="{{ image_name }}" build-date="{{ build_date }}"
{% endblock %}

{% block prometheus_openstack_exporter_header %}{% endblock %}

{% block prometheus_openstack_exporter_repository_version %}
ENV prometheus_openstack_exporter_version=1.3.0
{% endblock %}

{% block prometheus_openstack_exporter_install %}
RUN mkdir -p /opt/openstack-exporter
COPY openstack-exporter /opt/openstack-exporter/openstack-exporter
{% endblock %}

{% block prometheus_openstack_exporter_footer %}{% endblock %}
{% block footer %}{% endblock %}

USER prometheus
```

然后docker kolla image `python3 tools/build.py -b centos --type source prometheus-openstack-exporter --openstack-release ussuri`

最后到环境上试，结果没跑起来，错误如下：

```shell
/usr/local/bin/kolla_start: line 24: /opt/openstack-exporter/openstack-exporter: cannot execute binary file: Exec format error
```

网上查了一下，这个应该是因为编译环境不同，因为我在mac上build的，而我们最终跑在linux上。为了偷懒。。

然后我又在linux上装了一下golang

```shell
wget https://golang.org/dl/go1.15.7.linux-amd64.tar.gz
tar -C /usr/local -zxvf  go1.15.7.linux-amd64.tar.gz
```

编辑`/etc/profile`，增加两行到最后

```shell
export GOROOT=/usr/local/go
export PATH=$PATH:$GOROOT/bin
```

`source /etc/profile`即时生效，然后在继续go build，打包，上环境

然后问题解决啦，现在openstack exporter可以成功获取到数据啦

