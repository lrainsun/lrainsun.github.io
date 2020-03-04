---
layout: post
title:  "grafana一些作图小技巧"
date:   2020-03-04 23:00:00 +0800
categories: Prometheus
tags: Prometheus-Server
excerpt: grafana一些作图小技巧
mathjax: true
typora-root-url: ../
---

# Boom Table

## 活用pattern & query

多条查询语句返回多个对象多个值，反应在同一张表格里，这个时候可以用Boom Table

![image-20200304143954327](/assets/images/image-20200304143954327.png)

每一行是一条查询

![image-20200304144027291](/assets/images/image-20200304144027291.png)

我们通过Legend "TENANT"匹配pattern，Row Name和Col Name可以自定义

![image-20200304144205511](/assets/images/image-20200304144205511.png)

因为query的查询结果就只有一个value，我们想要获取到更多的信息，可以通过查询返回的label来取。但是label不会直接带到pattern里，如果我们想要在pattern里用到（很有可能会用到的，比如增加link，比如增加显示）

比如我们一条查询语句返回：

```shell
up{datacenter="AMS01",deployment="ci22am",instance="ci22amcmp004.webex.com:9183",job="openstack_exporter",ocp_deployment_type="datastack",ocp_release="2.7.1906.1"}  1
```

可以通过Legend定义`DEPLOYMENT.{{datacenter}}.{{deployment}}.{{ocp_deployment_type}}.{{ocp_release}}`

![image-20200304153807302](/assets/images/image-20200304153807302.png)

这样，label里四个数据都被带到pattern里去了。那pattern里怎么取呢？

`_1_` 就是datacenter，在这里是AMS01

`_2_` 就是deployment，在这里是ci22am

`_3_` 就是ocp_deployment_type，在这里是datastack

`_4_` 就是ocp_release，在这里是2.7.1906.1

怎么用呢？

比如pattern要定义col name的时候就可以用

![image-20200304154037726](/assets/images/image-20200304154037726.png)

就说明对这条记录，col name是ci22am

比如定义cell link

![image-20200304154113658](/assets/images/image-20200304154113658.png)

`dashboard/db/ocp-deployment-level-metrics?orgId=1&var-datasource=datasource-prometheus&var-datacenter=_1_&var-deployment=_2_&var-deploymenttype=_3_&var-release=_4_._5_._6_._7_`

对这条记录，link就是``dashboard/db/ocp-deployment-level-metrics?orgId=1&var-datasource=datasource-prometheus&var-datacenter=AMS01&var-deployment=ci22am&var-deploymenttype=datastack&var-release=2.7.1906.1`

## 巧算Total

![image-20200304154903119](/assets/images/image-20200304154903119.png)

最后这一列total，其实每个数字都是一条单独的查询，把total的值算出来，还是通过pattern放到表格对应位置上去

![image-20200304155843374](/assets/images/image-20200304155843374.png)

![image-20200304155905453](/assets/images/image-20200304155905453.png)

