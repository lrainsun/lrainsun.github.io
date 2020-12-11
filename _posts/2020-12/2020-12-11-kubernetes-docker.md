---
layout: post
title:  "Kubernetes跟docker的集成"
date:   2020-12-11 22:00:00 +0800
categories: Kubernetes
tags: Kubernetes-kubelet
excerpt: Kubernetes跟docker的集成
mathjax: true
typora-root-url: ../
---

# Kubernetes跟docker的集成

## CRI

容器运行时插件（Container Runtime Interface，简称 CRI）是 Kubernetes v1.5 引入的容器运行时接口，它将 Kubelet 与容器运行时解耦，将原来完全面向 Pod 级别的内部接口拆分成面向 Sandbox 和 Container 的 gRPC 接口，并将镜像管理和容器管理分离到不同的服务。

CRI 主要定义了两个 grpc interface.

- `RuntimeService`：容器(container) 和 (Pod)Sandbox 运行时管理
- `ImageService`：拉取、查看、和移除镜像

![image-20201211170123236](/../assets/images/image-20201211170123236.png)

![image-20201211170155560](/../assets/images/image-20201211170155560.png)

## docker cri

![image-20201211170715146](/../assets/images/image-20201211170715146.png)

dockershim之前是内置的kubelet里的

1. 第一步kubelet作为cri client通过grpc调用dockershim（cri server）
2. dockershim收到请求，转换成docker daemon能理解的请求发给docker daemon
3. docker daemon收到请求，把对容器的操作转发给containerd
4. containerd收到请求，创建一个containerd-shim的进程，去操作容器。只要创建一个容器，就会对应创建一个containerd-shim进程，这个进程是容器进程的父进程，维持stdin等fd打开等工作。containerd-shim跟containerd并没有父子关系，这样containerd意外退出的时候不会影响到容器
5. containerd-shim调用容器运行时，docker用的是runc，来启动容器
6. runc在启动完容器之后就退出了，containerd-shim会一直在，收集容器进程状态，上报给containerd

## CRI-Containerd

从 containerd 1.0 开始，为了能够减少一层调用的开销，containerd 开发了一个新的 daemon，叫做 CRI-Containerd，直接与 containerd 通信，从而取代了 dockershim

![img](/../assets/images/1334952-20190610155603613-1830747977.png)

## containerd

从 containerd 1.1 开始，社区选择在 containerd 中直接内建 CRI plugin，通过方法调用来进行交互，从而减少一层 gRPC 的开销

![img](/../assets/images/1334952-20190610155613625-585993135-20201211195821572.png)

## cri-o

但在 containerd 做这些事情之情，社区就已经有了一个更为专注的 cri-runtime：CRI-O，它非常纯粹，就是兼容 CRI 和 OCI，做一个 Kubernetes 专用的运行时

![img](/../assets/images/1334952-20190610155624697-374335939.png)

其中 conmon 就对应 containerd-shim，大体意图是一样的。

## 常见 CRI runtime 实现

![image-20201211200010214](/../assets/images/image-20201211200010214.png)

## dockershim

https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md#deprecation

从kubernetes1.20版本开始，dockershim就被标记成deprecation，计划大概在1.22版本（2021年晚些时候）移除，1.23版本就不支持dockershim了，那正在使用dockershim的用户就需要考虑使用其他的cri runtimes，比如containerd或者cri-o。

dockerfile还可以用，docker image也还可以用（docker image是标准的oci image），但是不需要再安装docker了。

# References

[1] [https://cloud.tencent.com/developer/article/1579900](https://cloud.tencent.com/developer/article/1579900)

[2] [https://kubernetes.io/blog/2020/12/02/dont-panic-kubernetes-and-docker/](https://kubernetes.io/blog/2020/12/02/dont-panic-kubernetes-and-docker/)

