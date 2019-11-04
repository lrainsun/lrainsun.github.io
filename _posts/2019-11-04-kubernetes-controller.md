---
layout: post
title:  "Kubernetes控制器基本原理"
date:   2019-11-04 22:03:00 +0800
categories: Kubernetes
tags: Kubernetes-Controller
excerpt: Kubernetes控制器的基本原理和实现
mathjax: true
typora-root-url: ../
---

# 控制器模型

在机器人设计和自动化的应用中，**控制循环**是一个用来调节系统状态的非终止循环。而在 Kubernetes 中，控制器就是前面提到的控制循环，它通过 API Server 监控整个集群的状态，并确保集群处于预期的工作状态。

控制器是由kube-controller-manager这个组件实现的，这个组件，就是一系列控制器的集合，这个目录下面的每一个控制器，都以独有的方式负责某种编排功能。控制器是基于 Go 语言的 [client-go](https://github.com/kubernetes/client-go) 库来实现的。

![image-20191104095615805](/assets/images/image-20191104095615805-2834854.png)

这些控制器之所以被统一放在 pkg/controller 目录下，就是因为它们都遵循 Kubernetes项目中的一个通用编排模式，即：控制循环（control loop）。
比如，现在有一种待编排的对象 X，它有一个对应的控制器。下面是伪代码描述的控制循环：

```
for {
	实际状态 := 获取集群中对象 X 的实际状态（Actual State）
	期望状态 := 获取集群中对象 X 的期望状态（Desired State）
	if 实际状态 == 期望状态{
		什么都不做
	} else {
		执行编排动作，将实际状态调整为期望状态
	}
}
```

Kubernetes 控制器会监视资源的创建/更新/删除事件，并触发 `Reconcile` 函数作为响应。整个调整过程被称作 “Reconcile Loop”（调谐循环）或者 “Sync Loop”（同步循环）。`Reconcile` 是一个使用 object（Resource 的实例）的命名空间和 object 名来调用的函数，使 object 的实际状态与 object 的 `Spec` 中定义的状态保持一致。调用完成后，`Reconcile` 会将 object 的状态更新为当前实际状态。

在具体实现中，实际状态往往来自于 Kubernetes 集群本身。
比如，kubelet 通过心跳汇报的容器状态和节点状态，或者监控系统中保存的应用监控数据，或者控制器主动收集的它自己感兴趣的信息，这些都是常见的实际状态的来源。
而期望状态，一般来自于用户提交的 YAML 文件。

![image-20191104102207257](/assets/images/image-20191104102207257.png)

类似 Deployment 这样的一个控制器，实际上都是由上半部分的控制器定义（包括期
望状态），加上下半部分的被控制对象的模板组成的。

# 工作原理

![image-20191104102048662](/assets/images/image-20191104102048662.png)

那控制器究竟是怎么实现这个Reconcile的工作，使得实际状态跟期望状态达到一致的呢？

每个控制器内部都有两个核心组件：`Informer/SharedInformer` 和 `Workqueue`。其中 `Informer/SharedInformer` 负责 watch Kubernetes 资源对象的状态变化，然后将相关事件（events）发送到 `Workqueue` 中，最后再由控制器的 `worker` 从 `Workqueue` 中取出事件交给控制器处理程序进行处理。

> **事件** = **动作**（create, update 或 delete） + **资源的 key**（以 `namespace/name` 的形式表示）

## Informer

控制器的主要作用是 watch 资源对象的当前状态和期望状态，然后发送指令来调整当前状态，使之更接近期望状态。**为了获得资源对象当前状态的详细信息，控制器需要向 API Server 发送请求。**

这个操作，依靠的是一个叫作 Informer（可以翻译为：通知器）的代码库完成的。Informer 与API 对象是一一对应的，比如Deployment对象就应该有一个对应的DeploymentInformer。Informer是一个自带缓存和索引机制，可以触发handler的客户端库，这个本地缓存在kubernetes中一般被称为store，索引一般被称为index。

### Reflector

在构造Informer的过程中，需要传入一个Client，Informer使用这个Client跟APIServer建立连接。而负责维护这个连接的，是Informer所使用的Reflector包。**Reflector 使用的是一种叫作ListAndWatch的方法，来“获取”并“监听”对象实例的变化。**

> 频繁地调用 API Server 非常消耗集群资源，因此为了能够多次 `get` 和 `list` 对象，Kubernetes 开发人员最终决定使用 `client-go` 库提供的缓存机制。控制器并不需要频繁调用 API Server，只有当资源对象被创建，修改或删除时，才需要获取相关事件。**`client-go` 库提供了 `Listwatcher` 接口用来获得某种资源的全部 Object，缓存在内存中；然后，调用 Watch API 去 watch 这种资源，去维护这份缓存；最后就不再调用 Kubernetes 的任何 API**

#### ListWatcher

`ListWatcher` 是对某个特定命名空间中某个特定资源的 `list` 和 `watch` 函数的集合。这样做有助于控制器只专注于某种特定资源。`fieldSelector` 是一种过滤器，它用来缩小资源搜索的范围，让控制器只检索匹配特定字段的资源。

```go
cache.ListWatch {
	listFunc := func(options metav1.ListOptions) (runtime.Object, error) {
		return client.Get().
			Namespace(namespace).
			Resource(resource).
			VersionedParams(&options, metav1.ParameterCodec).
			FieldsSelectorParam(fieldSelector).
			Do().
			Get()
	}
	watchFunc := func(options metav1.ListOptions) (watch.Interface, error) {
		options.Watch = true
		return client.Get().
			Namespace(namespace).
			Resource(resource).
			VersionedParams(&options, metav1.ParameterCodec).
			FieldsSelectorParam(fieldSelector).
			Watch()
	}
}
```

### Delta FIFO

在 ListAndWatch 机制下，一旦 APIServer 端有新的实例被创建、删除或者更新，Reflector 都会收到“事件通知”。这时，该事件及它对应的 API 对象这个组合，就被称为增量（Delta），它会被放进一个 Delta FIFO Queue（即：增量先进先出队列）中。

### Store

而另一方面，Informer 会不断地从这个 Delta FIFO Queue 里读取（Pop）增量。每拿到一个增量，Informer 就会判断这个增量里的事件类型，然后创建或者更新本地对象的缓存。这个缓存，在 Kubernetes 里一般被叫作 Store。

### Indexer

如果事件类型是 Added（添加对象），那么 Informer 就会通过一个叫作 Indexer 的库把这个增量里的 API 对象保存在本地缓存中，并为它创建索引。相反地，如果增量的事件类型是 Deleted（删除对象），那么 Informer 就会从本地缓存中删除这个对象。

Informer的主要职责有两个：

1. 同步本地缓存的工作，是 Informer 的第一个职责，也是它最重要的职责。
2. 第二个职责，根据这些事件的类型，触发事先注册好的ResourceEventHandler。这些Handler，需要在创建控制器的时候注册给它对应的Informer。

#### Resource Event Handler

`Resource Event Handler` 用来处理相关资源发生变化的事件：

```go
type ResourceEventHandlerFuncs struct {
	AddFunc    func(obj interface{})
	UpdateFunc func(oldObj, newObj interface{})
	DeleteFunc func(obj interface{})
}
```

- **AddFunc** : 当资源创建时被调用
- **UpdateFunc** : 当已经存在的资源被修改时就会调用 `UpdateFunc`。`oldObj` 表示资源的最近一次已知状态。如果 Informer 向 API Server 重新同步，则不管资源有没有发生更改，都会调用 `UpdateFunc`。
- **DeleteFunc** : 当已经存在的资源被删除时就会调用 `DeleteFunc`。该函数会获取资源的最近一次已知状态，如果无法获取，就会得到一个类型为 `DeletedFinalStateUnknown` 的对象。

#### ResyncPeriod

`ResyncPeriod` 用来设置控制器遍历缓存中的资源以及执行 `UpdateFunc` 的频率。这样做可以周期性地验证资源的当前状态是否与期望状态匹。每经过 resyncPeriod 指定的时间，Informer 维护的本地缓存，都会使用最近一次 LIST 返回的结果强制更新一次，从而保证缓存的有效性。

如果控制器错过了 update 操作或者上一次操作失败了，`ResyncPeriod` 将会起到很大的弥补作用。如果你想编写自定义控制器，不要把周期设置太短，否则系统负载会非常高。

这个定时 resync 操作，也会触发 Informer 注册的“更新”事件。但此时，这个“更新”事件对应的 Network 对象实际上并没有发生变化，即：新、旧两个对象的 ResourceVersion 是一样的。在这种情况下，Informer 就不需要对这个更新事件再做进一步的处理了。所以这里可以增加新、旧两个对象的版本（ResourceVersion）是否发生了变化，如果有变化才开始进行的入队操作。

### SharedInformer

Informer 会将资源缓存在本地以供自己后续使用。但 Kubernetes 中运行了很多控制器，有很多资源需要管理，难免会出现以下这种重叠的情况：一个资源受到多个控制器管理。

为了应对这种场景，可以通过 `SharedInformer` 来创建一份供多个控制器共享的缓存。这样就不需要再重复缓存资源，也减少了系统的内存开销。使用了 `SharedInformer` 之后，不管有多少个控制器同时读取事件，`SharedInformer` 只会调用一个 Watch API 来 watch 上游的 API Server，大大降低了 API Server 的负载。实际上 `kube-controller-manager` 就是这么工作的。

`SharedInformer` 提供 hooks 来接收添加、更新或删除某个资源的事件通知。还提供了相关函数用于访问共享缓存并确定何时启用缓存，这样可以减少与 API Server 的连接次数，降低 API Server 的重复序列化成本和控制器的重复反序列化成本。

```go
lw := cache.NewListWatchFromClient(…)
sharedInformer := cache.NewSharedInformer(lw, &api.Pod{}, resyncPeriod)
```

## WorkQueue

工作队列的作用是，负责同步 Informer 和控制循环之间的数据。

由于 `SharedInformer` 提供的缓存是共享的，所以它无法跟踪每个控制器，这就需要控制器自己实现排队和重试机制。因此，大多数 `Resource Event Handler` 所做的工作只是将事件放入消费者工作队列中。

每当资源被修改时，`Resource Event Handler` 就会放入一个 key 到 `Workqueue` 中。key 的表示形式为 `<resource_namespace>/<resource_name>`，如果提供了 `<resource_namespace>`，key 的表示形式就是 `<resource_name>`。每个事件都以 key 作为标识，因此每个消费者（控制器）都可以使用 workers 从 Workqueue 中读取 key。所有的读取动作都是串行的，这就保证了不会出现两个 worker 同时读取同一个 key 的情况。

每个key在workqueue中的生命周期：

![img](/assets/images/eENdLY.jpg)

如果处理事件失败，控制器就会调用 `AddRateLimited()` 函数将事件的 key 放回 `Workqueue` 以供后续重试（如果重试次数没有达到上限）。如果处理成功，控制器就会调用 `Forget()` 函数将事件的 key 从 `Workqueue` 中移除。**注意：该函数仅仅只是让 `Workqueue` 停止跟踪事件历史，如果想从 `Workqueue` 中完全移除事件，需要调用 `Done()` 函数。**

## 控制器

**控制器应该何时启用 workers 来处理 `Workqueue` 中的事件呢？**控制器需要等到缓存完全同步到最新状态才能开始处理 `Workqueue` 中的事件，主要有两个原因：

1. 在缓存完全同步之前，获取的资源信息是不准确的。
2. 对单个资源的多次快速更新将由缓存合并到最新版本中，因此控制器必须等到缓存变为空闲状态才能开始处理事件，不然只会把时间浪费在等待上。

这种做法的伪代码如下：

```go
controller.informer = cache.NewSharedInformer(...)
controller.queue = workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())

controller.informer.Run(stopCh)

if !cache.WaitForCacheSync(stopCh, controller.HasSynched)
{
	log.Errorf("Timed out waiting for caches to sync"))
}

// Now start processing
controller.runWorker()
```

所以，启动控制循环的逻辑非常简单：

1. 等待 Informer 完成一次本地缓存的数据同步操作；
2. 直接通过 goroutine 启动一个（或者并发启动多个）“无限循环”的任务。“无限循环”任务的每一个循环周期，执行的正是我们真正关心的业务逻辑。

![img](/assets/images/2019-06-29-033816.jpg)

# References

[1] [https://www.yangcs.net/posts/a-deep-dive-into-kubernetes-controllers/](https://www.yangcs.net/posts/a-deep-dive-into-kubernetes-controllers/)