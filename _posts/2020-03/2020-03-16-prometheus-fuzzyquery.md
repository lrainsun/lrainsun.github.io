---
layout: post
title:  "prometheus正则查询"
date:   2020-03-16 23:00:00 +0800
categories: Prometheus
tags: Prometheus-Server
excerpt: prometheus正则查询
mathjax: true
typora-root-url: ../
---

# variables

最近一直忙着做ocp report dashboard，通过一个centralized exporter-prometheus把所有dc的数据收集起来，通过grafana展示出来

grafana dashboard真的是很需要耐心。。。眼神一飘，数据搞不好就错了

搞得作业几天没做了，硬凑个数吧，有些东西长期不坚持就容易放弃了，凑完继续dashboard去。。。

每个grafana dashboard都可以定义一些variables，这些variables可以直接用到dashboard的图中。

比如通过这样一个query，可以取到所有的datacenter数据

![image-20200316195518843](/../assets/images/image-20200316195518843.png)

在其他地方就可以用`$datacenter`来使用

定义好之后，还可以在Preview的地方预览一下，数据对不对

![image-20200316195655233](/../assets/images/image-20200316195655233.png)

想要关联起来的话，就可以这样定义

![image-20200316195731079](/../assets/images/image-20200316195731079.png)

当你选中datacenter之后，就只有这个datacenter的所有deployment type被列出来

# 正则查询

可为啥，我们刚刚定义了之后，选择了datacenter，deployment type这里却是none呢？

![image-20200316195905711](/../assets/images/image-20200316195905711.png)

因为我们查询的时候没有用模糊查询，所以当选择两个datacenter的时候，就变成查询`datacenter="AMS01|DFW02"`了， 这样是没有结果的，因为没有一个datacenter的名字是`“AMS01|DFW02"`

需要`datacenter=~"AMS01|DFW02"`

![image-20200316200150301](/../assets/images/image-20200316200150301.png)

`=~'` 和`!~`表示支持正则表达式的查询

