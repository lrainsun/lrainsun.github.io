---
layout: post
title:  "Kubernetes Pod基本概念"
date:   2019-10-27
categories: Kubernetes
tags: Kubernetes-Pod
excerpt: Kubernetes最基本的概念Pod
mathjax: true
---

好记性不如烂笔头，尤其我这种，比较笨，脑子又不好使，看过就忘的，记点笔记帮助自己吧~

<br />

怎么理解Pod的概念呢？

* 可以简单把Pod理解为vm，而container则是vm里的用户进程
* 但是一个Pod里，通常应该仅运行一个应用
  * 不同的应用可以被调度到不同的主机，增加了资源利用率
  * 同一应用可按需进行规模变动，增强了系统架构的灵活性

<br />

Pod是kubernetes里最基本的概念：

* Pod是一个逻辑概念

  * 并不真正存在Pod这样一个隔离环境
  * 处理的是宿主机操作系统上的Linux容器的Namespace和Cgroups

  <br />

* Pod是 Kubernetes 项目的原子调度单位

  * Pod里可以包含多个container
  * Pod里的container共享同一Network（UTS, IPC...) Namespace
  * Pod里的container共享同一组数据卷

  <br />

* Pod的实现需要一个中间容器（Infra容器）

  * Infra容器是一个永远处于“暂停”状态的容器
  * Infra容器的目的是为了hold住Network Namespace
    * 开发网络插件时，应该考虑如何配置这个Pod的Network Namespace，而不是每个用户容器如何使用网络配置
  * Pod的生命周期跟Infra容器一致

  <br />

* Pod只有一个IP地址

  * 这个地址是Pod的Network Namespace对应的IP地址
  * Pod中的容器可以使用localhost进行通信
  * Pod中的容器看到的网络设备跟Infra容器看到的完全一样

  <br />

***我们为什么需要Pod？***

1. ***密切相关的容器可以放在同一个Pod里，统一调度***
2. ***Pod的意义在于容器设计模式的实现***（如Sidecar pattern, Ambassador pattern, Adapter Pattern, etc...)