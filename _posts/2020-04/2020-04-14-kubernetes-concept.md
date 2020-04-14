---
layout: post
title:  "kubernetes的本质"
date:   2020-04-14 23:00:00 +0800
categories: Kubernetes
tags: Kubernetes-Controller
excerpt: kubernetes的本质
mathjax: true
typora-root-url: ../
---

# kubernetes的本质

很久没看kubernetes都要忘了，复习一下：

kubernetes最大的特点是：声明式 API

而Kubernetes 项目的本质，是为用户提供一个具有普遍意义的容器编排工具

# 基本架构

![image-20200414201750796](/../assets/images/image-20200414201750796.png)

* controller node：由三个紧密协作的独立组件组合而成，它们分别是负责 API 服

  务的 kube-apiserver、负责调度的 kube-scheduler，以及负责容器编排的 kube-controllermanager

  。整个集群的持久化数据，则由 kube-apiserver 处理后保存在 Ectd 中。

* worker node：最初要的组件是kubelet

  * kubelet 负责同容器运行时CRI（Container Runtime Interface）打交道。这个接口定义了

    容器运行时的各项核心操作，比如：启动一个容器需要的所有参数。

    而具体的容器运行时，比如 Docker 项目，则一般通过 OCI 这个容器运行时规范同底层的 Linux 操

    作系统进行交互，即：把 CRI 请求翻译成对 Linux 操作系统的调用（操作 Linux Namespace 和

    Cgroups 等）

  * kubelet 还通过 gRPC 协议同一个叫作 Device Plugin 的插件进行交互。这个插件，是

    Kubernetes 项目用来管理 GPU 等宿主机物理设备的主要组件

  * kubelet通过CNI（Container Networking Interface）调用网络插件为容器配置网络

  * kubelet通过CSI（Container Storage Interface）存储插件为容器配置持久化存储

# 解决问题

按照用户的意愿和整个系统的规则，完全自动化地处理好容器之间的各种关系 —— 容器编排

![image-20200414203158736](/../assets/images/image-20200414203158736.png)

紧密交互关系的container（应用之间需要非常频繁的交互和访问，或通过本地文件进行信息交换） ——> Pod ——> 一次启动一个/多个应用的实例（副本） ——> Deployment（StatefulSet, DaemonSet, Job, CronJob） ——> 这一组Pod需要一个固定IP地址和端口以负荷均衡方式访问 ——> Service ——>  从外部访问集群内部服务的入口 ——> Ingress

Pod访问需要有授权信息 ——> Secret；Pod运行需要配置文件 ——> ConfigMap

Pod自动水平扩展 ——>  HorizontalPodAutoScaler