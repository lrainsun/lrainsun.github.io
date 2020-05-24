---
layout: post
title:  "Kubernetes有状态应用复习"
date:   2020-05-24 23:00:00 +0800
categories: Kubernetes
tags: Kubernetes-Controller
excerpt: Kubernetes有状态应用复习
mathjax: true
typora-root-url: ../
---

# Kubernetes有状态应用复习

纯复习篇

假如一个应用中多个实例中间的运行顺序是有规定的，比如一定要先A再B，要怎么做？

## headless service

headless service的定义就是service spec里的`clusterIP: None`，含义是这样，当用service dns访问service的时候，分为两种情况

* 如果有定义clusterIP，那么访问这个service DNS `my-svc.my-namespace.svc.cluster.local`时解析到的就是这个service的vip
* 如果没有定义clusterIP，那么访问这个service DNS `my-svc.my-namespace.svc.cluster.local`时解析到是这个service后端某个pod的IP
  * 而每一个service代理的pod的ip，也都会被绑定一个dns记录，`my-pod.my-svc.my-namespace.svc.cluster.local`

## statefulset

一个statefuleset spec中定义的replicas数目，跟pod name是有关系的：

比如replicas：3，name是test的话，那么就会建立三个pod，分别为：

`test-0`, `test-1`,` test-2`

那么对应的dns就是`test-0.my-svc.my-namespace.svc.cluster.local`,`test-1.my-svc.my-namespace.svc.cluster.local`, `test-2.my-svc.my-namespace.svc.cluster.local`

statefuleset可以保证：

* 对于pod的创建，是严格按照编号的顺序进行的，在前一个pod进到running(ready)之前，后一个pod会处于pending状态
* 即使pod被删除，仍然会按照原来编号的顺序创建出新的pod，并分配跟原来相同的网络身份，比如test-0

虽然dns记录本身不会变，但是解析到的pod地址可能会改变，并不固定。所以对于有状态应用实例的访问，需要用dns记录或者hostname的方式，而不能用pod的ip

## PV/PVC

与此同时，如果在statefulset中定义pvc（volumeClaimTemplates），那么被这个statefuleset管理的pod，都会声明一个对应的PVC，而跟pod name一样，这个pvc的名字也会被分配一个一致的编号`pvc-name.statefulset-name-编号`

比如`test-0`绑定`pvc.test-0`,`test-1`绑定`pvc.test-1`,这个顺序也是固定的，即使pod被删除，最终创建的pod也仍然会跟同样编号的pvc进行绑定

# 总结

StatefulSet 其实就是一种特殊的Deployment，而其独特之处在于，它的每个 Pod 都被编号了。而且，这个编号会体现在 Pod的名字和 hostname 等标识信息上，这不仅代表了 Pod 的创建顺序，也是 Pod 的重要网络标识（即：在整个集群里唯一的、可被的访问身份）。有了这个编号后，StatefulSet 就使用 Kubernetes 里的两个标准功能：Headless Service 和PV/PVC，实现了对 Pod 的拓扑状态和存储状态的维护。