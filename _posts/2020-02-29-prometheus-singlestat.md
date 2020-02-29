---
layout: post
title:  "grafana singlestat时间的秘密"
date:   2020-02-29 23:00:00 +0800
categories: Prometheus
tags: Prometheus-Server
excerpt: grafana singlestat时间的秘密
mathjax: true
typora-root-url: ../
---

# SingleStat

![image-20200229231045173](/assets/images/image-20200229231045173.png)

如果我们在query设置的是instant，意味着取的是当前timestamp的值，所以无论右上角的时间range怎么变，这个值是不会改变的

反之如果不是instant，这里就是一个取值的range，随着右上角的变化，整个图会变

![image-20200229231207761](/assets/images/image-20200229231207761.png)

那么中间这个值哪来的呢？

![image-20200229231320055](/assets/images/image-20200229231320055.png)

设置中有个stat，这里设置的是average，也就是会把time range内的采样值求平均

在我们这个case中，用current更为合适