---
layout: post
title:  "Kubernetes CRI"
date:   2019-11-17 21:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Architecture
excerpt: Kubernetes CRI
mathjax: true
typora-root-url: ../
---

# kubelet & CRI

kubernetes在调度好之后，最终将pod在宿主机上创建出来，并把各个容器启动起来，都是kubelet做的事情，所以kubelet是很重要的组件。

## kubelet工作原理

![image-20191117173502863](/assets/images/image-20191117173502863.png)

kubelet 本身，也是按照“控制器”模式来工作的。

kubelet 的工作核心，就是一个控制循环，即：SyncLoop（图中的大圆圈）。而驱动这个控制循环运行的事件，包括四种：

1. Pod 更新事件；
2. Pod 生命周期变化；
3.  kubelet 本身设置的执行周期；
4.  定时的清理事件。

跟其他控制器类似，kubelet 启动的时候，要做的第一件事情，就是设置 Listers，也就是注册它所关心的各种事件的 Informer。这些 Informer，就是 SyncLoop 需要处理的数据的来源。

此外，kubelet 还负责维护着很多很多其他的子控制循环（也就是图中的小圆圈）。这些控制循环的名字，一般被称作某某 Manager，比如 Volume Manager、Image Manager、Node Status Manager 等等。

同样kubelet 也是通过 Watch 机制，监听了与自己相关的 Pod 对象的变化。当然，这个Watch 的过滤条件是该 Pod 的 nodeName 字段与自己相同。kubelet 会把这些 Pod 的信息缓存在自己的内存里。kubelet 会启动一个名叫 Pod Update Worker 的、单独的Goroutine 来完成对 Pod 的处理工作。

## CRI

kubelet主要的功能就是启动和停止容器，称之为容器运行时（Container Runtime），kubernetes从1.5开始加入了容器运行时插件API，也就是container runtime interface，简称CRI。

kubelet在调用下层容器运行时的执行过程，并不会直接调用docker API，而是通过CRI的gRPC接口来间接执行。之所以要在kubelet中引入一层单独的抽象，是为了对 Kubernetes屏蔽下层容器运行时的差异。

# CRI主要组件

kubelet使用gRPC框架通过unix socket与CRI代理进行通信，在这个过程中kubelet是客户端，CRI代理（shim）是服务端。

CRI的主要组件，Protocol Buffers API包含两个gRPC服务：

* ImageService：提供从仓库拉取镜像，查看和移除镜像的功能
* RuntimeService：负责Pod和容器的生命周期管理，以及容器的交互（exec, attach, port-forward）

docker & rkt都可以使用一个socket同时提供这两个服务，在kubelet中可以用--container-runtime-endpoint和--image-service-endpoint设置这个socket。

![image-20191117174320572](/assets/images/image-20191117174320572.png)

## Pod和容器生命周期的管理

Pod由一组应用容器组成，其中包含共有的环境和资源约束。在CRI里，这个环境被称为PodSandBox。每个容器运行时都可以有自己的实现方式，对于hypervisor类的运行时，PodSandBox会具体化为一个虚拟机；对于docker，会是一个linux命名空间。v1alph1 API中，kubelet会创建pod级别的cgroup传递给容器运行时，并以此运行所有进程来满足PodSandBox对pod的资源保障。

启动Pod之前，kubelet调用RuntimeService.RunPodSandbox来创建环境。这一过程为Pod设置网络资源（分配IP等操作），PadSandbox被激活之后，就可以独立创建，启动，停止和删除不同的容器了。kubelet会在停止和删除PodSandbox之前首先停止和删除其中的容器。

kubelet负责通过RPC管理容器的生命周期，实现容器生命周期的钩子，存活和健康监测，以及执行Pod的重启策略等。

RuntimeService包括对Sandbox和Container操作的方法。

**虽然kubernetes的最小调度单元式Pod，但是，CRI是容器级别的实现**。