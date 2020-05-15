---
layout: post
title:  "Kubernetes自定义控制器"
date:   2019-12-02 22:03:00 +0800
categories: Kubernetes
tags: Kubernetes-Controller
excerpt: Kubernetes自定义控制器的依赖库
mathjax: true
typora-root-url: ../
---

# client-go

kube-controller-manager的实现是依赖于client-go的，之前学习的controller的实现都是基于client-go

![img](/../assets/images/1.png)

图中的组件分为client-go和custom controller两部分：

1. client-go部分
   - Reflector: 监视特定资源的k8s api, 把新监测的对象放入Delta Fifo队列，完成此操作的函数是ListAndWatch。
   - Informer: 从Delta Fifo队列拿出对象，完成此操作的函数是processLoop。
   - Indexer: 提供线程级别安全来存储对象和key。
2. custom-controller部分
   - Informer reference: Informer对象引用
   - Indexer reference: Indexer对象引用
   - Resource Event Handlers: 被Informer调用的回调函数，这些函数的作用通常是获取对象的key，并把key放入Work queue，以进一步做处理。
   - Work queue: 工作队列，用于将对象的交付与其处理分离，编写Resource event handler functions以提取传递的对象的key并将其添加到工作队列。
   - Process Item: 用于处理Work queue中的对象，可以有一个或多个其他函数一起处理；这些函数通常使用Indexer reference或Listing wrapper来检索与该键对应的对象。

# controller-runtime

仿照kube-controller-manager去写controller是一个思路，或者，有一个工具kubebuilder，它将这些controller的逻辑封装和抽象成了公共的库（controller-runtime）和公共的工具（controller-tools），可以帮助我们自动生成scaffold的代码来初始化crd，封装底层的go-client

![image-20191202134442946](/../assets/images/image-20191202134442946.png)

## Clients/Cache

在实现 Controller 的时候不可避免地需要对某些资源类型进行创建/删除/更新，就是通过该 Clients 实现的，其中查询功能实际查询是本地的 Cache，写操作直接访问 Api Server。

## Scheme

每一组 Controllers 都需要一个 Scheme，提供了 Kinds 与对应 Go types 的映射。

比如说我们给定一个 Scheme: "tutotial.kubebuilder.io/api/v1".CronJob{} 这个 Go type 映射到 batch.tutotial.kubebuilder.io/v1 的 CronJob GVK（GroupVersionKind），那么从 Api Server 获取到下面的 JSON:

```yaml
{

"kind": "CronJob",

"apiVersion": "batch.tutorial.kubebuilder.io/v1",

...}
```

就能构造出对应的 Go type了，通过这个 Go type 也能正确地获取 GVR 的一些信息，控制器可以通过该 Go type 获取到期望状态以及其他辅助信息进行调谐逻辑。

## controller

Kubebuidler 为我们生成scaffold代码，我们只需要实现 Reconcile 方法即可。

## Index

由于 Controller 经常要对 Cache 进行查询，Kubebuilder 提供 Index utility 给 Cache 加索引提升查询效率。

## Finalizer

* 对象 ObjectMeta 里面的 Finalizers 不为空
  * 对该对象的 delete 操作就会转变为 update 操作，update deletionTimestamp 字段，其意义就是告诉 K8s 的 GC“在deletionTimestamp 这个时刻之后
* Finalizers 为空
  * 立马删除掉该对象

在创建对象时把 Finalizers 设置好（任意 string），然后处理 DeletionTimestamp 不为空的 update 操作（实际是 delete），根据 Finalizers 的值执行完所有的 pre-delete hook（此时可以在 Cache 里面读取到被删除对象的任何信息）之后将 Finalizers 置为空即可。

## OwnerReference

K8s GC 在删除一个对象时，任何 ownerReference 是该对象的对象都会被清除，与此同时，Kubebuidler 支持所有对象的变更都会触发 Owner 对象 controller 的 Reconcile 方法。

# Reference

[1] [http://www.iceyao.com.cn/2019/01/14/Kubernetes%E7%BC%96%E5%86%99%E8%87%AA%E5%AE%9A%E4%B9%89controller/](http://www.iceyao.com.cn/2019/01/14/Kubernetes%E7%BC%96%E5%86%99%E8%87%AA%E5%AE%9A%E4%B9%89controller/)

[2] [https://juejin.im/post/5d8acac2e51d4577f54a0fd2](https://juejin.im/post/5d8acac2e51d4577f54a0fd2)

