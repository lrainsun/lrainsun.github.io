---
layout: post
title:  "Cluster API"
date:   2019-12-03 15:03:00 +0800
categories: Kubernetes
tags: Kubernetes-Controller
excerpt: Cluster API
mathjax: true
typora-root-url: ../
---

# Cluster-API

Cluster-API是一个部署kubernetes的工具，kubernetes集群可能运行在任何环境上，openstack, aws, vmware, baremetal等等，Cluster-API可以帮助我们在这些环境上把基础设施build好，然后再通过kubeadm来部署cluster。

而Cluster-API本身也是运行在kubernetes上的，所以就需要一个bootstrap cluster用来跑cluster-api。

具体流程如下：

* 在bootstrap cluster中部署CRD，以及cluster-api控制器和provider控制器
* 在bootstrap cluster中创建资源，如Cluster，Machine等，cluster-api的控制器会自动创建好虚拟机
* 虚拟机创建好之后，会通过kubeadm创建kubernetes集群

# 实现

[cluster-api](https://github.com/kubernetes-sigs/cluster-api) 的业务逻辑实现也在controller的syncLoop，跟kubebuilder一样，也依赖了controller-runtime的库。

在main函数之前，init函数（init会先于main函数执行，实现包级别的一些初始化操作）里会先创建scheme，并把cluster-api会用到的clientgoscheme，clusterv1alpha2，clusterv1alpha3，kubeadmbootstrapv1 AddToScheme，这样cache就知道watch谁了

```go
var (
   scheme   = runtime.NewScheme()
   setupLog = ctrl.Log.WithName("setup")
)

func init() {
   klog.InitFlags(nil)

   _ = clientgoscheme.AddToScheme(scheme)
   _ = clusterv1alpha2.AddToScheme(scheme)
   _ = clusterv1alpha3.AddToScheme(scheme)
   _ = kubeadmbootstrapv1.AddToScheme(scheme)
   // +kubebuilder:scaffold:scheme
}
```

main函数入口一开始会初始化一些变量，然后会NewManager创建一个manager，作为多个controller的载体。main里会做的事情都是围绕Manager来的：

* 初始化一个Manager
* 将 Manager 的 Client 传给 Controller，并且调用 SetupWithManager 方法传入 Manager 进行 Controller 的初始化
* 启动 Manager

刚刚初始化好的scheme，会作为参数传递到mgr

```go
mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
   Scheme:             scheme,
   MetricsBindAddress: metricsAddr,
   LeaderElection:     enableLeaderElection,
   LeaderElectionID:   "controller-leader-election-capi",
   Namespace:          watchNamespace,
   SyncPeriod:         &syncPeriod,
   NewClient:          newClientFunc,
   Port:               webhookPort,
})
```

在new一个Manager的时候，setOptionsDefaults里会做一些赋值的动作

```go
// Set default values for options fields
options = setOptionsDefaults(options)

// setOptionsDefaults set default values for Options fields
func setOptionsDefaults(options Options) Options {
	// Use the Kubernetes client-go scheme if none is specified
	if options.Scheme == nil {
		options.Scheme = scheme.Scheme
	}

	if options.MapperProvider == nil {
		options.MapperProvider = func(c *rest.Config) (meta.RESTMapper, error) {
			return apiutil.NewDynamicRESTMapper(c)
		}
	}

	// Allow newClient to be mocked
	if options.NewClient == nil {
		options.NewClient = defaultNewClient
	}

	// Allow newCache to be mocked
	if options.NewCache == nil {
		options.NewCache = cache.New
	}

	// Allow newRecorderProvider to be mocked
	if options.newRecorderProvider == nil {
		options.newRecorderProvider = internalrecorder.NewProvider
	}

	// Allow newResourceLock to be mocked
	if options.newResourceLock == nil {
		options.newResourceLock = leaderelection.NewResourceLock
	}

	if options.newMetricsListener == nil {
		options.newMetricsListener = metrics.NewListener
	}
	leaseDuration, renewDeadline, retryPeriod := defaultLeaseDuration, defaultRenewDeadline, defaultRetryPeriod
	if options.LeaseDuration == nil {
		options.LeaseDuration = &leaseDuration
	}

	if options.RenewDeadline == nil {
		options.RenewDeadline = &renewDeadline
	}

	if options.RetryPeriod == nil {
		options.RetryPeriod = &retryPeriod
	}

	if options.EventBroadcaster == nil {
		options.EventBroadcaster = record.NewBroadcaster()
	}

	return options
}
```

然后回到Manager的New函数，这里，首先会创建cache，`options.NewCache = cache.New`，所以其实就是调用`cache.New`来创建cache。这时候创建了InformersMap，Scheme 里面的每个 GVK 都创建了对应的 Informer，通过 informersByGVK 这个 map 做 GVK 到 Informer 的映射，每个 Informer 会根据 ListWatch 函数对对应的 GVK 进行 List 和 Watch。

```go
// Create the cache for the cached read client and registering informers
cache, err := options.NewCache(config, cache.Options{Scheme: options.Scheme, Mapper: mapper, Resync: options.SyncPeriod, Namespace: options.Namespace})
if err != nil {
   return nil, err
}

// New initializes and returns a new Cache.
func New(config *rest.Config, opts Options) (Cache, error) {
	opts, err := defaultOpts(config, opts)
	if err != nil {
		return nil, err
	}
	im := internal.NewInformersMap(config, opts.Scheme, opts.Mapper, *opts.Resync, opts.Namespace)
	return &informerCache{InformersMap: im}, nil
}

// newSpecificInformersMap returns a new specificInformersMap (like
// the generical InformersMap, except that it doesn't implement WaitForCacheSync).
func newSpecificInformersMap(config *rest.Config,
	scheme *runtime.Scheme,
	mapper meta.RESTMapper,
	resync time.Duration,
	namespace string,
	createListWatcher createListWatcherFunc) *specificInformersMap {
	ip := &specificInformersMap{
		config:            config,
		Scheme:            scheme,
		mapper:            mapper,
		informersByGVK:    make(map[schema.GroupVersionKind]*MapEntry),
		codecs:            serializer.NewCodecFactory(scheme),
		paramCodec:        runtime.NewParameterCodec(scheme),
		resync:            resync,
		createListWatcher: createListWatcher,
		namespace:         namespace,
	}
	return ip
}
```

apireader：通过提供的config & options创建一个client，直接跟server进行交互，这里是没有cache的，写操作会用这个

apiwriter：这边传入了上面创建的cache，`options.NewCache = cache.New`，读操作会使用这个cache

```go
apiReader, err := client.New(config, client.Options{Scheme: options.Scheme, Mapper: mapper})
if err != nil {
   return nil, err
}

writeObj, err := options.NewClient(cache, config, client.Options{Scheme: options.Scheme, Mapper: mapper})
if err != nil {
   return nil, err
}
```

Metrics

```go
// Create the metrics listener. This will throw an error if the metrics bind
// address is invalid or already in use.
metricsListener, err := options.newMetricsListener(options.MetricsBindAddress)
if err != nil {
   return nil, err
}
```

controller的初始化，这边用的是builder模式，为Cluster这个对象创建一个controller，前面这一串`.`都是在给Build传入参

```go
if err = (&controllers.ClusterReconciler{
   Client: mgr.GetClient(),
   Log:    ctrl.Log.WithName("controllers").WithName("Cluster"),
}).SetupWithManager(mgr, concurrency(clusterConcurrency)); err != nil {
   setupLog.Error(err, "unable to create controller", "controller", "Cluster")
   os.Exit(1)
}

func (r *ClusterReconciler) SetupWithManager(mgr ctrl.Manager, options controller.Options) error {
	c, err := ctrl.NewControllerManagedBy(mgr).
		For(&clusterv1.Cluster{}).
		Watches(
			&source.Kind{Type: &clusterv1.Machine{}},
			&handler.EnqueueRequestsFromMapFunc{ToRequests: handler.ToRequestsFunc(r.controlPlaneMachineToCluster)},
		).
		WithOptions(options).
		Build(r)

	if err != nil {
		return errors.Wrap(err, "failed setting up with a controller manager")
	}

	r.controller = c
	r.recorder = mgr.GetEventRecorderFor("cluster-controller")

	return nil
}
```

实际的build在这里，需要doController，doWatch

```go
// Build builds the Application ControllerManagedBy and returns the Controller it created.
func (blder *Builder) Build(r reconcile.Reconciler) (controller.Controller, error) {
   if r == nil {
      return nil, fmt.Errorf("must provide a non-nil Reconciler")
   }
   if blder.mgr == nil {
      return nil, fmt.Errorf("must provide a non-nil Manager")
   }

   // Set the Config
   blder.loadRestConfig()

   // Set the ControllerManagedBy
   if err := blder.doController(r); err != nil {
      return nil, err
   }

   // Set the Watch
   if err := blder.doWatch(); err != nil {
      return nil, err
   }

   return blder.ctrl, nil
}
```

doController，初始化了一个controller

```go
func (blder *Builder) doController(r reconcile.Reconciler) error {
   name, err := blder.getControllerName()
   if err != nil {
      return err
   }
   ctrlOptions := blder.ctrlOptions
   ctrlOptions.Reconciler = r
   blder.ctrl, err = newController(name, blder.mgr, ctrlOptions)
   return err
}

// New returns a new Controller registered with the Manager.  The Manager will ensure that shared Caches have
// been synced before the Controller is Started.
func New(name string, mgr manager.Manager, options Options) (Controller, error) {
	if options.Reconciler == nil {
		return nil, fmt.Errorf("must specify Reconciler")
	}

	if len(name) == 0 {
		return nil, fmt.Errorf("must specify Name for Controller")
	}

	if options.MaxConcurrentReconciles <= 0 {
		options.MaxConcurrentReconciles = 1
	}

	// Inject dependencies into Reconciler
	if err := mgr.SetFields(options.Reconciler); err != nil {
		return nil, err
	}

	// Create controller with dependencies set
	c := &controller.Controller{
		Do:                      options.Reconciler,   //Reconcile逻辑
		Cache:                   mgr.GetCache(),       //对象的缓存，通过informer接收对象的事件，为缓存中的对象添加索引
		Config:                  mgr.GetConfig(),
		Scheme:                  mgr.GetScheme(),
		Client:                  mgr.GetClient(),      //对kubernetes资源进行crud
		Recorder:                mgr.GetEventRecorderFor(name),   //事件收集
		Queue:                   workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), name), //watch资源的cud事件缓存
		MaxConcurrentReconciles: options.MaxConcurrentReconciles,
		Name:                    name,
	}

	// Add the controller as a Manager components
	return c, mgr.Add(c)
}
```

doWatch，watch刚刚传入的apiType是Cluster的对象，同时对managedObjects也进行watch

```go
func (blder *Builder) doWatch() error {
   // Reconcile type
   src := &source.Kind{Type: blder.apiType}
   hdler := &handler.EnqueueRequestForObject{} //EnqueueRequestForObject enqueue一个包含了对象name&namespace的request，包括create, update, delete
   err := blder.ctrl.Watch(src, hdler, blder.predicates...)
   if err != nil {
      return err
   }

   // Watches the managed types
   for _, obj := range blder.managedObjects {
      src := &source.Kind{Type: obj}
      hdler := &handler.EnqueueRequestForOwner{
         OwnerType:    blder.apiType,
         IsController: true,
      }
      if err := blder.ctrl.Watch(src, hdler, blder.predicates...); err != nil {
         return err
      }
   }

   // Do the watch requests
   for _, w := range blder.watchRequest {
      if err := blder.ctrl.Watch(w.src, w.eventhandler, blder.predicates...); err != nil {
         return err
      }
   }
   return nil
}
```

mgr启动

```go
setupLog.Info("starting manager")
if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil { //starts all registered Controllers and blocks until the Stop channel is closed
   setupLog.Error(err, "problem running manager")
   os.Exit(1)
}
```

```go
func (cm *controllerManager) Start(stop <-chan struct{}) error {
   // join the passed-in stop channel as an upstream feeding into cm.internalStopper
   defer close(cm.internalStopper)

   // Metrics should be served whether the controller is leader or not.
   // (If we don't serve metrics for non-leaders, prometheus will still scrape
   // the pod but will get a connection refused)
   if cm.metricsListener != nil {
      go cm.serveMetrics(cm.internalStop)
   }

   go cm.startNonLeaderElectionRunnables()

   if cm.resourceLock != nil {
      err := cm.startLeaderElection()
      if err != nil {
         return err
      }
   } else {
      go cm.startLeaderElectionRunnables()
   }

   select {
   case <-stop:
      // We are done
      return nil
   case err := <-cm.errChan:
      // Error starting a controller
      return err
   }
}

func (cm *controllerManager) startNonLeaderElectionRunnables() {
	cm.mu.Lock()
	defer cm.mu.Unlock()

	cm.waitForCache()

	// Start the non-leaderelection Runnables after the cache has synced
	for _, c := range cm.nonLeaderElectionRunnables {
		// Controllers block, but we want to return an error if any have an error starting.
		// Write any Start errors to a channel so we can return them
		ctrl := c
		go func() {
			cm.errChan <- ctrl.Start(cm.internalStop)
		}()
	}
}
```

waitForCache里，会start cache：Cache 的初始化核心是初始化所有的 Informer，Informer 的初始化核心是创建了 reflector 和内部 controller，reflector 负责监听 Api Server 上指定的 GVK，将变更写入 delta 队列中，可以理解为变更事件的生产者，内部 controller 是变更事件的消费者，他会负责更新本地 indexer，以及计算出 CUD 事件推给我们之前注册的 Watch Handler。

```go
func (cm *controllerManager) waitForCache() {
   if cm.started {
      return
   }

   // Start the Cache. Allow the function to start the cache to be mocked out for testing
   if cm.startCache == nil {
      cm.startCache = cm.cache.Start
   }
   go func() {
      if err := cm.startCache(cm.internalStop); err != nil {
         cm.errChan <- err
      }
   }()

   // Wait for the caches to sync.
   // TODO(community): Check the return value and write a test
   cm.cache.WaitForCacheSync(cm.internalStop)
   cm.started = true
}

// Start calls Run on each of the informers and sets started to true.  Blocks on the stop channel.
func (m *InformersMap) Start(stop <-chan struct{}) error {
	go m.structured.Start(stop)
	go m.unstructured.Start(stop)
	<-stop
	return nil
}

func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
	defer utilruntime.HandleCrash()

	fifo := NewDeltaFIFO(MetaNamespaceKeyFunc, s.indexer)

	cfg := &Config{
		Queue:            fifo,
		ListerWatcher:    s.listerWatcher,
		ObjectType:       s.objectType,
		FullResyncPeriod: s.resyncCheckPeriod,
		RetryOnError:     false,
		ShouldResync:     s.processor.shouldResync,

		Process: s.HandleDeltas,
	}

	func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()

		s.controller = New(cfg)
		s.controller.(*controller).clock = s.clock
		s.started = true
	}()

	// Separate stop channel because Processor should be stopped strictly after controller
	processorStopCh := make(chan struct{})
	var wg wait.Group
	defer wg.Wait()              // Wait for Processor to stop
	defer close(processorStopCh) // Tell Processor to stop
	wg.StartWithChannel(processorStopCh, s.cacheMutationDetector.Run)
	wg.StartWithChannel(processorStopCh, s.processor.run)

	defer func() {
		s.startedLock.Lock()
		defer s.startedLock.Unlock()
		s.stopped = true // Don't want any new listeners
	}()
	s.controller.Run(stopCh)
}
```

controller启动goroutine 不断地查询队列，如果有变更消息则触发到自定义的 Reconcile 逻辑。

所以最重要的业务逻辑实现就在reconcile的实现中

# Reference

[1] [http://www.iceyao.com.cn/2019/01/14/Kubernetes%E7%BC%96%E5%86%99%E8%87%AA%E5%AE%9A%E4%B9%89controller/](http://www.iceyao.com.cn/2019/01/14/Kubernetes%E7%BC%96%E5%86%99%E8%87%AA%E5%AE%9A%E4%B9%89controller/)

[2] [https://juejin.im/post/5d8acac2e51d4577f54a0fd2](https://juejin.im/post/5d8acac2e51d4577f54a0fd2)

