---
layout: post
title:  "Kubelet中的PLEG"
date:   2020-03-07 23:00:00 +0800
categories: Kubernetes
tags: Kubernetes-Kubelet
excerpt: Kubelet中的PLEG
mathjax: true
typora-root-url: ../
---

# PLEG

PLEG(Pod Lifecycle Event Generator)是kubelet的核心模块

The PLEG module in kubelet (Kubernetes) adjusts the container runtime state with each matched pod-level event and keeps the pod cache up to date by applying changes.

PLEG维护了一个缓存，缓存中记录了Pod的信息，通过记录的Pod信息，识别出Pod生命周期中的各种事件，如容器的启动、终止等，从runtime获取containers/sandboxes的信息，并根据前后两次信息对比，生成对应的PodLifecycleEvent，通过eventChannel发送到kubelet syncLoop进行消费，最终由kubelet syncPod完成Pod的同步。

![img](/../assets/images/orig-pleg-1.png)

PLEG会随着kubelet启动，其实是启动一个goroutine，每个relistPeriod(plegRelistPeriod = time.Second * 1，1s)就调用一次relist，根据最新的PodStatus生成PodLiftCycleEvent。

relist是PLEG的核心：

* podRecords：relist调用g.runtime.GetPods(true)获取所有的pods，生成 podRecords
* Old podRecord：上一轮的podRecord
* Current podRecord：这一轮的podRecord
* relist会根据old podRecord和current podRecord，检查pod中所有的container，分析出容器的启动，退出事件，然后把事件记录到缓存中
* 记录下来的事件最后会被发送到eventChannel

eventChannel的消费者是kubelet syncLoop，也就是说，每次kubelet sync都会检查pod状态并做更新。

# References

[1] [https://developers.redhat.com/blog/2019/11/13/pod-lifecycle-event-generator-understanding-the-pleg-is-not-healthy-issue-in-kubernetes/](https://developers.redhat.com/blog/2019/11/13/pod-lifecycle-event-generator-understanding-the-pleg-is-not-healthy-issue-in-kubernetes/)

[2] [https://cloud.tencent.com/developer/article/1359236](https://cloud.tencent.com/developer/article/1359236)

[3] [https://www.jianshu.com/p/7940d1a68062]( https://www.jianshu.com/p/7940d1a68062)

