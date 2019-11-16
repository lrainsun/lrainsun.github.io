---
layout: post
title:  "Kubernetes Operator"
date:   2019-11-16 23:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Controller
excerpt: Kubernetes Operator
mathjax: true
typora-root-url: ../
---

# Operator

对于有状态应用，除了statefulset，还有另外一个解决方案就是Operator。Operator是用来扩展Kubernetes API，特定的应用程序控制器，它用来创建、配置和管理复杂的有状态应用，如数据库、缓存和监控系统。Operator基于Kubernetes的资源和控制器概念之上构建，但同时又包含了应用程序特定的领域知识。创建Operator的关键是CRD（自定义资源）的设计。

# 工作原理

Operator本质上，其实是一组CRD，以及对CRD的控制器的实现。

因为业务的多样性，kubernetes内置的控制器类型未必能够很好满足所有的业务需求，有些时候可能就需要自定义资源，并自己对这些资源进行管理控制。有两部分内容:

* [CRD](https://lrainsun.github.io/2019/11/05/kubernetes-crd/)
  * 对业务资源的定义在CRD里，需要很清晰地通过CRD把你的业务需要（资源的描述，以及desired state）表达出来。
* [CRD controller](https://lrainsun.github.io/2019/11/04/kubernetes-controller/) 
  * 怎样从当前状态能够最终达到desired state，这中间主要的业务逻辑包含在CRD的控制器部分，需要代码来实现。参考控制器的实现，通常是采用listwatch的方式，当有变更或者跟期望状态不符的时候，作出相应的处理。

Operator 和 StatefulSet 并不是竞争关系。你完全可以编写一个 Operator，然后在Operator 的控制循环里创建和控制 StatefulSet 而不是 Pod。比如，业界知名的Prometheus项目的 Operator，正是这么实现的。
此外，CoreOS 公司在被 RedHat 公司收购之后，已经把 Operator 的编写过程封装成了一个叫作Operator SDK的工具（整个项目叫作 Operator Framework），它可以帮助你生成Operator 的框架代码。