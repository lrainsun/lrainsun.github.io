---
layout: post
title:  "Kubernetes调度"
date:   2019-12-18 22:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Scheduler
excerpt: Kubernetes调度
mathjax: true
typora-root-url: ../
---

# kubernetes默认调度器

调度器（scheduler）的职责，是为一个新创建出来的Pod，寻找一个最合适的节点（Node）。跟openstack的调度（filter & weight）策略异曲同工，也是分为两步：

1. 从集群所有的节点中，根据调度算法挑选出所有可以运行该 Pod 的节点（Predicate）

   调用一组叫作 Predicate 的调度算法，来检查每个Node

2. 从第一步的结果中，再根据调度算法挑选一个最符合条件的节点作为最终结果（Priority）

   再调用一组叫作 Priority 的调度算法，来给上一步得到的结果里的每个 Node 打分，调度的结果是得分最高的Node

调度成功后，scheduler会把Pod的Spec.nodeName字段填上目标节点的名字（所以如果你在创建Pod的时候自己填了这个字段，就可以骗过scheduler以为已经被调度了呢）

# 工作原理

![image-20191218214454042](/../assets/images/image-20191218214454042.png)

从图中可以看出，Kubernetes 的调度器的核心，实际上是两个相互独立的控制循环。

## Informer Path

它的主要目的，是启动一系列Informer，用来监听（Watch）Etcd 中 Pod、Node、Service 等与调度相关的 API 对象的变化。比如，当一个待调度 Pod（即：它的 nodeName 字段是空的）被创建出来之后，调度器就会通过 Pod Informer 的 Handler，将这个待调度 Pod 添加进调度队列。

Kubernetes 的调度队列是一个 PriorityQueue（优先级队列），并且当某些集群信息发生变化的时候，调度器还会对调度队列里的内容进行一些特殊操作。

Kubernetes 的默认调度器还要负责对调度器缓存（即：scheduler cache）进行更新。Kubernetes 调度部分进行性能优化的一个最根本原则，就是尽最大可能将集群信息Cache 化，以便从根本上提高 Predicate 和 Priority 调度算法的执行效率。

## Scheduling Path

Scheduling Path 的主要逻辑，就是不断地从调度队列里出队一个 Pod。然后，调用Predicates 算法进行“过滤”。这一步“过滤”得到的一组 Node，就是所有可以运行这个Pod 的宿主机列表。当然，Predicates 算法需要的 Node 信息，都是从 Scheduler Cache 里直接拿到的，这是调度器保证算法执行效率的主要手段之一。

“Cache 化”，这个变化其实是最近几年 Kubernetes 调度器性能得以提升的一个关键演化。

接下来，调度器就会再调用 Priorities 算法为上述列表里的 Node 打分，分数从 0 到 10。得分最高的 Node，就会作为这次调度的结果。

调度算法执行完成后，调度器就需要将 Pod 对象的 nodeName 字段的值，修改为上述 Node的名字。这个步骤在 Kubernetes 里面被称作 Bind。

为了不在关键调度路径里远程访问 APIServer，Kubernetes 的默认调度器在 Bind 阶段，只会更新 Scheduler Cache 里的 Pod 和 Node 的信息。这种基于“乐观”假设的 API 对象更新方式，在 Kubernetes 里被称作Assume。

Assume 之后，调度器才会创建一个 Goroutine 来异步地向 APIServer 发起更新 Pod 的请求，来真正完成 Bind 操作。如果这次异步的 Bind 过程失败了，其实也没有太大关系，等Scheduler Cache 同步之后一切就会恢复正常。

由于上述 Kubernetes 调度器的“乐观”绑定的设计，当一个新的 Pod 完成调度需要在某个节点上运行起来之前，该节点上的 kubelet 还会通过一个叫作 Admit 的操作来再次验证该 Pod 是否确实能够运行在该节点上。这一步 Admit 操作，实际上就是把一组叫作GeneralPredicates 的、最基本的调度算法，比如：“资源是否可用”“端口是否冲突”等再执行一遍，作为 kubelet 端的二次确认。

## 无锁化

在 Scheduling Path 上，调度器会启动多个 Goroutine 以节点为粒度并发执行 Predicates 算法，从而提高这一阶段的执行效率。而与之类似的，Priorities 算法也会以 MapReduce 的方式并行计算然后再进行汇总。而在这些所有需要并发的路径上，调度器会避免设置任何全局的竞争资源，从而免去了使用锁进行同步带来的巨大的性能损耗。
所以，在这种思想的指导下，如果你再去查看一下前面的调度器原理图，你就会发现，Kubernetes 调度器只有对调度队列和 Scheduler Cache 进行操作时，才需要加锁。而这两部分操作，都不在 Scheduling Path 的算法执行路径上。

# 可扩展机制

默认调度期的可扩展性：采用go plugin机制，每一个绿色的箭头都是一个可以插入自定义逻辑的接口

![image-20191218221320395](/../assets/images/image-20191218221320395.png)