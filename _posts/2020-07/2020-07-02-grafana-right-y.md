---
layout: post
title:  "grafana graph里的y坐标"
date:   2020-07-02 23:00:00 +0800
categories: Prometheus
tags: Prometheus-Server
excerpt: grafana graph里的y坐标
mathjax: true
typora-root-url: ../
---

# grafana graph里的y坐标

当我们画grafana图的时候，有时候想把两个内容画到一起，觉得是相关的内容，但是呢，两者的单位或者啥的有没有相关性

这个使用可以用两个y坐标来表示

![image-20200702172212859](/../assets/images/image-20200702172212859.png)

像这样，左边的y坐标表示了打开文件标识符，右边的y坐标表示了上下文切换，两者的单位是不同的，尽管这样，还是可以很清楚区分

![image-20200702172336156](/../assets/images/image-20200702172336156.png)

像这样，我们有两个metrics

![image-20200702172427221](/../assets/images/image-20200702172427221.png)

通过匹配legend的方式，可以把switches定义到右边的y坐标

![image-20200702172552691](/../assets/images/image-20200702172552691.png)

label可以分别定义两个y坐标的名称，更加清晰

![image-20200702172701679](/../assets/images/image-20200702172701679.png)