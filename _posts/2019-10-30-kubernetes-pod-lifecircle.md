---
layout: post
title:  "Kubernetes Pod生命周期"
date:   2019-10-30 21:50:00 +0800
categories: Kubernetes
tags: Kubernetes-Pod
excerpt: Kubernetes Pod生命周期
mathjax: true
typora-root-url: ../
---

上一次有学习kubernetes pod最基本的概念，今天回顾一下Pod的生命周期。

Pod对象自从其创建开始，一直到它终止退出的时间范围称为Pod的生命周期。

# Pod phase

在这段时间中，Pod会处于多种不同的状态，在生命周期中，Pod总是处于生命周期中以下几个phase之一：

* Pending：API Server创建了Pod资源对象并已存入ETCD中，但他尚未被调度完成，或者仍处于从registry下载image的过程中
* Running：Pod已经被调度至某节点，并且所有的容器都已经被kubelet创建完成
* Succeeded：Pod中的所有容器都已经成功终止并且不会重启
* Failed：所有容器都已经终止，但至少有一个容器终止失败，即容器返回了非0值的退出状态或者已经被系统终止
* Unknown：API Server无法正常获取到Pod对象的状态信息，通常是由于其无法与所在工作节点的kubelet通信所致

# Pod Status

Pod有个PodStatus对象，其中包含一个 [PodCondition](https://kubernetes.io/docs/resources-reference/v1.7/#podcondition-v1-core) 数组。 PodCondition 用于描述造成当前status的具体原因是什么。可能的值有 PodScheduled、Ready、Initialized 和 Unschedulable。

# Pod生命周期

在生命周期中Pod会执行一些操作，这些操作有些是必须（M）的，有些是可选（O）的，而操作是否执行取决于Pod的定义：

* create main container（M）
* run init container（O）
* post start hook（O）
* liveness probe（O）
* readiness probe（O）
* pre stop hook（O）

![img](/assets/images/loap-20191030211604014.png)

1. 图里面没有画出来的是，在所有这些之前，infra container先会被创建，用来hold住namespace，等待其他container join
2. 如果定义了init container，那么它将会最先被创建，通常init container用于为主容器执行一些预置操作
   1. init container (spec.initContainers) 必须运行完成直到结束，如果init container运行失败，那么kubernetes需要重启它直到成功完成（如果spec.restartPolicy不为"Never"）
   2. 可能会有多个init container，每个init container都必须按照定义的顺序串行运行
3. 然后main container和post-start hook会同时launch，post-start并不一定会在main container的ENTRYPOINT之前运行
4. 然后liveness probe和readiness probe会开始起作用，这两个如果没定义的话，默认是会返回success的（健康检查是由kubelet周期性执行的）
   1. liveness probe是存活性检测，如果这个检测不通过，kubelet会把container kill掉并根据restartPolicy决定是否将其重启
   2. readiness probe是就绪性检测，用于判断container是否准备就绪并可对外提供服务，没有通过过检测的话，会把其IP从pod对象service对象的端点列表中移除
5. 等到pod被kill的时候，pre-stop hook会先被执行，以同步的方式调用，所以在他完成之前会阻塞删除容器的操作的调用。这中间还有个grace period（spec.terminationGracePeriodSeconds），默认30s，这个时间超出后，如果pre-stop hook还没运行完，kubelet会发送SIGTERM并再等2秒
6. 最后main container会被删除，grace period超出后，kubernetes就选择SIGKILL强制kill掉Pod了



## Pod的创建流程

![image-20191030211642133](/assets/images/image-20191030211642133.png)

1. 用户通过kubectl或者其他API客户端提交Pod Spec给API Server
2. API Server尝试着将Pod对象的相关信息存入ETCD，待写入操作执行完成，API Server即会返回确认信息至客户端
3. API Server开始反映ETCD中的状态变化
4. 所有的Kubernetes组件均使用"watch"机制来跟踪检查API Server上的相关变动
5. kube-scheduler通过”watcher“觉察到API Server创建了新的Pod对象但尚未绑定至任何工作节点
6. kube-scheduler为Pod对象挑选一个工作节点并将结果信息更新至API Server
7. 调度结果信息由API Server更新至ETCD，而且API Server也开始反映此Pod对象的调度结果
8. Pod被调度到的worker node上的kubelet尝试在当前节点上调用docker启动容器，并将容器的结果状态返回给API Server
9. API Server将Pod状态信息存入ETCD系统
10. 在ETCD确认写入操作完成之后，API Server将确认信息发送至相关的kubelet，事件将通过它被接受

## Pod的终止过程

![image-20191030211707070](/assets/images/image-20191030211707070.png)

1. 用户发送删除Pod对象的命令
2. API Server中的对象会随着时间的推移而更新，在前面说过的grace period （默认30s），Pod被视为”dead"
3. 将Pod标记为“Terminating"状态
4. （与第三步同时运行）kubelete在监控到Pod对象转为"Terminating"状态的同时启动Pod关闭流程
5. （与第三步同时运行）Endpoint Controller在监控到Pod对象的关闭行为时将其从所有匹配到此endpoint的service资源的endpoint列表中移除
6. 如果当前Pod对象定义了pre-stop hook，则在其标记为"Terminating"后即会以同步的方式自动执行；如果grace period超时后，pre-stop还没执行完，则第二步会被重新执行并额外获取一个时长为2秒的小宽限期
7. Pod对象中的容器收到TERM信号
8. grace period超时后，若存在任何一个仍在运行的进程，那么Pod对象即会收到SIGKILL信号
9. kubelete请求API Server将此Pod资源的宽限期设置为0从而完成删除操作，他变得对用户不再可见
10. docker会删除pod中的container，数据会从ETCD被删除