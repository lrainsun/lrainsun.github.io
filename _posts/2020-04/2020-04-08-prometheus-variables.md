---
layout: post
title:  "grafana组合variables"
date:   2020-03-12 23:01:00 +0800
categories: Prometheus
tags: Prometheus-Server
excerpt: grafana组合variables
mathjax: true
typora-root-url: ../
---

# grafana组合variables

通常我们定义variables是这样，取一个query，把label_values取出来即可，像这样

```shell
label_values(ocp_exporter_deployment{job="ocp_exporter",datacenter=~"$datacenter",ocp_deployment_type=~"$ocp_deployment_type"}, deployment)
```

但有时候会有这种需求：

比如在我们的查询A里，包含有label1的两个资源；在我们的查询B里，包含有label2的三个资源

我们希望在variables可以list出来所有这五个资源，这个时候label_values就不能用了，可以这样

```shell
query_result(count by (hypervisor_name)(ocp_exporter_hypervisor_cpu_ratio{job="ocp_exporter",datacenter=~"$datacenter",ocp_deployment_type=~"$ocp_deployment_type",deployment=~"$deployment",aggregate=~"$aggregate" ,hypervisor_name=~".*cmp.*"}) or count by (hypervisor_name) (ocp_libvirt_exporter_instance{job="ocp_exporter",datacenter=~"$datacenter",ocp_deployment_type=~"$ocp_deployment_type",deployment=~"$deployment",aggregate=~"$aggregate"}))
```

用or把两个metric合并起来，用query_result

并且Regex这样配就可以了`/{hypervisor_name="(.*)"}/`

![image-20200408211456905](/../assets/images/image-20200408211456905.png)

