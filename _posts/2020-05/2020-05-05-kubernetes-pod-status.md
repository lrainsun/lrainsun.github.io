---
layout: post
title:  "Kubernetes Pod的状态"
date:   2020-05-05 23:00:00 +0800
categories: Kubernetes
tags: Kubernetes-Pod
excerpt: Kubernetes Pod的状态
mathjax: true

---

# Kubernetes Pod的状态

restartPolicy，配置了Pod 的恢复策略：

* Always：在任何情况下，只要容器不在运行状态，就自动重启容器；

* OnFailure: 只在容器 异常时才自动重启容器；

* Never: 从来不重启容器。

而一个pod里是可能有一个或者多个容器的，那么整个pod的状态，跟restartpolicy结合起来看就是这样的：

* 只要 Pod 的 restartPolicy 指定的策略允许重启异常的容器（比如：Always），那么这个 Pod就会保持 Running 状态，并进行容器重启。否则，Pod 就会进入 Failed 状态 。
* 对于包含多个容器的 Pod，只有它里面所有的容器都进入异常状态后，Pod 才会进入 Failed 状态。在此之前，Pod 都是 Running 状态。此时，Pod 的 READY 字段会显示正常容器的个数

## pending

Pending状态说明pod没有被调度到一个节点上，通常因为资源不足。可以describe一下

## waiting

waiting状态说明pod已经调度到节点上，但是没有运行起来。可能原因有拉取镜像失败。可以describe一下

## crashing或unhealthy

可以通过看容器log来查看具体问题

## running但没有正常工作

可以通过`kubectl create --valicate -f **.yaml`创建pod来检查错误是什么

# 探针

Kubelet 可以选择是否执行在容器上运行的三种探针执行和做出反应：

- `livenessProbe`：指示容器是否正在运行。如果存活探测失败，则 kubelet 会杀死容器，并且容器将受到其 [重启策略](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy) 的影响。如果容器不提供存活探针，则默认状态为 `Success`。
- `readinessProbe`：指示容器是否准备好服务请求。如果就绪探测失败，端点控制器将从与 Pod 匹配的所有 Service 的端点中删除该 Pod 的 IP 地址。初始延迟之前的就绪状态默认为 `Failure`。如果容器不提供就绪探针，则默认状态为 `Success`。
- `startupProbe`: 指示容器中的应用是否已经启动。如果提供了启动探测(startup probe)，则禁用所有其他探测，直到它成功为止。如果启动探测失败，kubelet 将杀死容器，容器服从其重启策略进行重启。如果容器没有提供启动探测，则默认状态为成功`Success`。