---
layout: post
title:  "grafana里的promQL"
date:   2020-02-28 23:00:00 +0800
categories: Prometheus
tags: Prometheus-Server
excerpt: grafana里的promQL
mathjax: true
typora-root-url: ../
---

# Grafana Table

初衷是想做一个表格，获取上一星期比如新增的server个数，那就需要要求在上星期结束的时间点取一下所有server的个数，在上星期开始的时间点取一下server的个数，减一下那就是差值了。

比如我取server个数的查询语句是`sum by (deployment) (sum by (deployment, hypervisor_hostname) (nova_servers{job="openstack_exporter"}))`

那么一个星期前是`sum by (deployment) (sum by (deployment, hypervisor_hostname) (nova_servers{job="openstack_exporter"} offset 1w)) `

但是我怎么让这个时间控制在上个星期呢？

![image-20200228215559422](/../assets/images/image-20200228215559422.png)

这边点一下prometheus，可以跳转到Prometheus查看时间

![image-20200228215731955](/../assets/images/image-20200228215731955.png)

而这时候grafana右上角的time range是

![image-20200228215804076](/../assets/images/image-20200228215804076.png)

时区关系，实际，`sum by (deployment) (sum by (deployment, hypervisor_hostname) (nova_servers{job="openstack_exporter"}))`是取了Time Range的From -> To中的`To`的时间

变更这个time range，再去验证一下，发现prometheus查询的时间是跟着变化的。那我们想要取到上一星期的值，就可以把time range设置成上星期的绝对时间

# Boom Table

另外，想要实现下面这样，一个表格中多个metrics，需要用到Boom Table，这个插件默认是不启用的，

![image-20200228220036385](/../assets/images/image-20200228220036385.png)

安装方法是

```shell
grafana-cli plugins install yesoreyeram-boomtable-panel
```

然后重启grafana

然后可以配置多个Metrics，每个Metrics的value值都可以被显示出来

![image-20200228220228129](/../assets/images/image-20200228220228129.png)

需要定义不同的Pattern匹配

![image-20200228220324157](/../assets/images/image-20200228220324157.png)

上面定义的Legend对应到Pattern和Col Name。比如这样一个Lagend `prod.server.my-app-01.sys.cpu.usage`

```shell
_4_                         --> cpu
_4_ _5_                     --> cpu usage
_4_ 2 _5_                   --> cpu 2 usage
_4_ use                     --> cpu use
Production _4_ usage        --> Production cpu usage
Production _4_ $somevar     --> Production cpu value_of_somevar_variable
_series_                    --> prod.server.my-app-01.sys.cpu.usage
_1_ _1_                     --> server server
_4_ __5_                    --> cpu _usage
```

