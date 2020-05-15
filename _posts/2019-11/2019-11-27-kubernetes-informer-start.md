---
layout: post
title:  "Kubernetes controller Informer启动代码分析"
date:   2019-11-27 23:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Controller
excerpt: Kubernetes controller Informer启动代码分析
mathjax: true
typora-root-url: ../
---


informer启动的流程，是从**SharedIndexInformer.Run**开始的

```go
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

来看看informer起来的流程：

* 先是创建了一个DeltaFIFO的队列

  ```go
  func NewDeltaFIFO(keyFunc KeyFunc, knownObjects KeyListerGetter) *DeltaFIFO {
     f := &DeltaFIFO{
        items:        map[string]Deltas{},
        queue:        []string{},
        keyFunc:      keyFunc,
        knownObjects: knownObjects,
     }
     f.cond.L = &f.lock
     return f
  }
  
  type DeltaFIFO struct {
  	// lock/cond protects access to 'items' and 'queue'.
  	lock sync.RWMutex
  	cond sync.Cond
  
  	// We depend on the property that items in the set are in
  	// the queue and vice versa, and that all Deltas in this
  	// map have at least one Delta.
  	items map[string]Deltas
  	queue []string
  
  	// populated is true if the first batch of items inserted by Replace() has been populated
  	// or Delete/Add/Update was called first.
  	populated bool
  	// initialPopulationCount is the number of items inserted by the first call of Replace()
  	initialPopulationCount int
  
  	// keyFunc is used to make the key used for queued item
  	// insertion and retrieval, and should be deterministic.
  	keyFunc KeyFunc
  
  	// knownObjects list keys that are "known", for the
  	// purpose of figuring out which items have been deleted
  	// when Replace() or Delete() is called.
  	knownObjects KeyListerGetter
  
  	// Indication the queue is closed.
  	// Used to indicate a queue is closed so a control loop can exit when a queue is empty.
  	// Currently, not used to gate any of CRED operations.
  	closed     bool
  	closedLock sync.Mutex
  }
  ```

  创建DeltaFIFO的时候有两个输入参数：

  * keyFunc是根据对象生成key的函数
  * knownObjects是一个本地缓存，可以看到传入的是informer的indexer

* 这里定义了一个Deltas处理的函数 HandleDeltas

  ```go
  func (s *sharedIndexInformer) HandleDeltas(obj interface{}) error {
     s.blockDeltas.Lock()
     defer s.blockDeltas.Unlock()
  
     // from oldest to newest
     for _, d := range obj.(Deltas) { //对每个deltas进行处理
        switch d.Type {
        case Sync, Added, Updated: //如果是sync, added or updated
           isSync := d.Type == Sync
           s.cacheMutationDetector.AddObject(d.Object)
           if old, exists, err := s.indexer.Get(d.Object); err == nil && exists {
              if err := s.indexer.Update(d.Object); err != nil {
                 return err
              }  //如果local store里已经存在，那么就update 
              s.processor.distribute(updateNotification{oldObj: old, newObj: d.Object}, isSync)
           } else {
              if err := s.indexer.Add(d.Object); err != nil {
                 return err
              }  //如果local store里没有，那么就add
              s.processor.distribute(addNotification{newObj: d.Object}, isSync)
           }
        case Deleted:
           if err := s.indexer.Delete(d.Object); err != nil {
              return err
           }
           s.processor.distribute(deleteNotification{oldObj: d.Object}, false)
        }
     }
     return nil
  }
  ```

  对于s.processor.distribute，先插一句，在创建controller的时候，比如replicaset，会AddEventHandler 

  ```go
  rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
     AddFunc:    rsc.enqueueReplicaSet,
     UpdateFunc: rsc.updateRS,
     // This will enter the sync loop and no-op, because the replica set has been deleted from the store.
     // Note that deleting a replica set immediately after scaling it to 0 will not work. The recommended
     // way of achieving this is by performing a `stop` operation on the replica set.
     DeleteFunc: rsc.enqueueReplicaSet,
  })
  rsc.rsLister = rsInformer.Lister()
  rsc.rsListerSynced = rsInformer.Informer().HasSynced
  
  podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
     AddFunc: rsc.addPod,
     // This invokes the ReplicaSet for every pod change, eg: host assignment. Though this might seem like
     // overkill the most frequent pod update is status, and the associated ReplicaSet will only list from
     // local storage, so it should be ok.
     UpdateFunc: rsc.updatePod,
     DeleteFunc: rsc.deletePod,
  })
  ```

  最终实现在AddEventHandlerWithResyncPeriod里，这里会把上面定义的resourceeventhandler都作为一个ProcessListener并加到process的Listener里

  ```go
  listener := newProcessListener(handler, resyncPeriod, determineResyncPeriod(resyncPeriod, s.resyncCheckPeriod), s.clock.Now(), initialBufferSize)
  
  if !s.started {
     s.processor.addListener(listener)
     return
  }
  ```

  然后回过头来看process.distribution，是把这三种notification根据具体情况分别加到每一个listener上去，也就是加到processorListener的addCh里。

  shared informer 是一堆informer的集合，每一类资源在一个客户端下只会有一个informer，如果其他的controller关注对应的资源，就注册一个对应的处理函数，当事件发生的时候，由shared informer统一进行事件的分发处理

  ![image-20191127204243909](/../assets/images/image-20191127204243909.png)

  *   ```
      func (p *sharedProcessor) distribute(obj interface{}, sync bool) {
         p.listenersLock.RLock()
         defer p.listenersLock.RUnlock()
      
         if sync {
            for _, listener := range p.syncingListeners {
               listener.add(obj)
            }
         } else {
            for _, listener := range p.listeners {
               listener.add(obj)
            }
         }
      }
      
      type updateNotification struct {
         oldObj interface{}
         newObj interface{}
      }
      
      type addNotification struct {
         newObj interface{}
      }
      
      type deleteNotification struct {
         oldObj interface{}
      }
      ```

* 然后会根据cfg创建一个sharedIndexInformer的controller，s.controller = New(cfg)

* 然后cacheMutationDetector.Run检查缓存对象是否变化，processor.run

* ```go
  wg.StartWithChannel(processorStopCh, s.cacheMutationDetector.Run)
  wg.StartWithChannel(processorStopCh, s.processor.run)
  ```

  * 对每个listener，都会开始listener.run & listener.pop

    ```go
    func (p *sharedProcessor) run(stopCh <-chan struct{}) {
       func() {
          p.listenersLock.RLock()
          defer p.listenersLock.RUnlock()
          for _, listener := range p.listeners {
             p.wg.Start(listener.run)
             p.wg.Start(listener.pop)
          }
          p.listenersStarted = true
       }()
       <-stopCh
       p.listenersLock.RLock()
       defer p.listenersLock.RUnlock()
       for _, listener := range p.listeners {
          close(listener.addCh) // Tell .pop() to stop. .pop() will tell .run() to stop
       }
       p.wg.Wait() // Wait for all .pop() and .run() to stop
    }
    ```

  * listener.run，从nextCh里取出每个notification进行处理

    ```go
    func (p *processorListener) run() {
       // this call blocks until the channel is closed.  When a panic happens during the notification
       // we will catch it, **the offending item will be skipped!**, and after a short delay (one second)
       // the next notification will be attempted.  This is usually better than the alternative of never
       // delivering again.
       stopCh := make(chan struct{})
       wait.Until(func() {
          // this gives us a few quick retries before a long pause and then a few more quick retries
          err := wait.ExponentialBackoff(retry.DefaultRetry, func() (bool, error) {
             for next := range p.nextCh {  //从nextCh里取出每一个notification进行处理
                switch notification := next.(type) {
                case updateNotification: //根据具体的情况，调用handler里的处理函数
                   p.handler.OnUpdate(notification.oldObj, notification.newObj)
                case addNotification:
                   p.handler.OnAdd(notification.newObj)
                case deleteNotification:
                   p.handler.OnDelete(notification.oldObj)
                default:
                   utilruntime.HandleError(fmt.Errorf("unrecognized notification: %T", next))
                }
             }
             // the only way to get here is if the p.nextCh is empty and closed
             return true, nil
          })
    
          // the only way to get here is if the p.nextCh is empty and closed
          if err == nil {
             close(stopCh)
          }
       }, 1*time.Minute, stopCh)
    }
    ```

  *  listener.pop

    ![image-20191127204402589](/assets/images/image-20191127204402589.png)

    ```go
    func (p *processorListener) pop() {
       defer utilruntime.HandleCrash()
       defer close(p.nextCh) // Tell .run() to stop
    
       var nextCh chan<- interface{}
       var notification interface{}
       for {
          select {  //select 随机执行一个可运行的 case。如果没有 case 可运行，它将阻塞，直到有 case 可运行
          case nextCh <- notification: //如果notifacation不为空，就把数据放入到nextCh中
             //第一次的时候，pendingNotifications读取数据会返回notification是nil
             //然后ok是false，nextCh置为nil，当channel为nil的时候，不接受任何数据，当前case条件不满足
             // Notification dispatched
             var ok bool
             notification, ok = p.pendingNotifications.ReadOne()
             // 从pendingNotifications中读取, 如果没有数据则把nextCh设置为空，disable这个select case
             if !ok { // Nothing to pop
                nextCh = nil // Disable this select case
             }
          case notificationToAdd, ok := <-p.addCh: //从addCh取出notification
             if !ok {
                return
             }
             if notification == nil { // No notification to pop (and pendingNotifications is empty) 没有notification了，pendingNotifications是空
                // Optimize the case - skip adding to pendingNotifications
                //上一次ReadOne的时候置成nil，然后继续读取数据，nextCh设置为p.nextCh，不为nil就可以接续接收数据
                notification = notificationToAdd
                nextCh = p.nextCh
             } else { // There is already a notification waiting to be dispatched
                //notification不为空，说明正在分发数据到nextCh中，就先将数据写入pendingNotifications（RingGrowing，如果队列容量不足就会进行扩容）中等待后续读取数据
                p.pendingNotifications.WriteOne(notificationToAdd)
             }
          }
       }
    }
    
    // pendingNotifications is an unbounded ring buffer that holds all notifications not yet distributed.
    pendingNotifications buffer.RingGrowing
    ```

    ![image-20191127204454188](/../assets/images/image-20191127204454188.png)

  nextCh是当前listener内部事件通道，而addCh负责接收sharedProessor传递进来的事件通知，pendingNotifications是内部缓冲队列。这样做的目的是在distribute的时候尽可能减少阻塞。

* 然后调用s.controller.Run

  ```go
  // Run begins processing items, and will continue until a value is sent down stopCh.
  // It's an error to call Run more than once.
  // Run blocks; call via go.
  func (c *controller) Run(stopCh <-chan struct{}) {
     defer utilruntime.HandleCrash()
     go func() {
        <-stopCh
        c.config.Queue.Close()
     }()
     r := NewReflector(
        c.config.ListerWatcher,
        c.config.ObjectType,
        c.config.Queue,
        c.config.FullResyncPeriod,
     )
     r.ShouldResync = c.config.ShouldResync
     r.clock = c.clock
  
     c.reflectorMutex.Lock()
     c.reflector = r
     c.reflectorMutex.Unlock()
  
     var wg wait.Group
     defer wg.Wait()
  
     wg.StartWithChannel(stopCh, r.Run)
  
     wait.Until(c.processLoop, time.Second, stopCh)
  }
  ```

  * controller Run的时候会初始化一个Reflector，入参包括包括ListerWatcher、ObjectType、Queue、FullResyncPeriod。Reflector从api server通过ListWatch“获取”并“监听”对象实例的变化，再放到Delta FIFO Queue的队列中去。

  * 然后会调用Reflector的Run

    ```go
    // Run starts a watch and handles watch events. Will restart the watch if it is closed.
    // Run will exit when stopCh is closed.
    func (r *Reflector) Run(stopCh <-chan struct{}) {
       klog.V(3).Infof("Starting reflector %v (%s) from %s", r.expectedTypeName, r.resyncPeriod, r.name)
      wait.Until(func() {  //每隔r.period的时间执行以下r.ListAndWatch (period:        time.Second)
          if err := r.ListAndWatch(stopCh); err != nil {
             utilruntime.HandleError(err)
          }
       }, r.period, stopCh)
    }
    
    // ListAndWatch first lists all items and get the resource version at the moment of call,
    // and then use the resource version to watch.
    // It returns error if ListAndWatch didn't even try to initialize watch.
    func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
    }
    ```

    ListAndWatch里会：

    * 获取所有对象的list并调用`DeltaFIFO.Replace(syncWith)`用`list`替代之前的元素
    * 启动一个异步goroutine，每隔r.resyncPeriod`时间调用`DeltaFIFO`的`Resync
    * 根据当前最新的`ResourceVersion`生成一个`watch`, 开始一直监控后面的变化

* 最后会调用c.processLoop，reflector向queue里面添加数据，processLoop会不停去消费这里这些数据

  ```go
  // processLoop drains the work queue.
  // TODO: Consider doing the processing in parallel. This will require a little thought
  // to make sure that we don't end up processing the same object multiple times
  // concurrently.
  //
  // TODO: Plumb through the stopCh here (and down to the queue) so that this can
  // actually exit when the controller is stopped. Or just give up on this stuff
  // ever being stoppable. Converting this whole package to use Context would
  // also be helpful.
  func (c *controller) processLoop() {
     for {
        obj, err := c.config.Queue.Pop(PopProcessFunc(c.config.Process))
        if err != nil {
           if err == ErrFIFOClosed {
              return
           }
           if c.config.RetryOnError {
              // This is the safe way to re-enqueue.
              c.config.Queue.AddIfNotPresent(obj)
           }
        }
     }
  }
  ```

  前面已经说了Process: s.HandleDeltas，所以这里就是调用s.HandleDeltas来处理。就是从DeltaFIFO Queue取出对象，然后distribute到Listener。

* 而每个Listener都有对应事件的hander函数，比如ReplicaSet：

  ```go
  podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
     AddFunc: rsc.addPod,
     // This invokes the ReplicaSet for every pod change, eg: host assignment. Though this might seem like
     // overkill the most frequent pod update is status, and the associated ReplicaSet will only list from
     // local storage, so it should be ok.
     UpdateFunc: rsc.updatePod,
     DeleteFunc: rsc.deletePod,
  })
  ```

  在addPod里，就会 `rsc.enqueueReplicaSet(rs)`，这里会`rsc.queue.Add(key)`这里的queue，就是workQueue。而[syncLoop](https://lrainsun.github.io/2019/11/24/kubernetes-relicaset-code/)，就会从这个workQueue取出数据来进行业务处理。

  ```go
  // Controllers that need to be synced
  queue workqueue.RateLimitingInterface
  ```

# Reference

[1] [http://www.sreguide.com/2019/10/14/kubernetes-shared-informer-%E7%B3%BB%E7%BB%9F%E8%AE%BE%E8%AE%A1%E4%B8%8E%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/](http://www.sreguide.com/2019/10/14/kubernetes-shared-informer-%E7%B3%BB%E7%BB%9F%E8%AE%BE%E8%AE%A1%E4%B8%8E%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)