---
layout: post
title:  "Kubernetes RelicaSetController代码简析"
date:   2019-11-24 21:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Controller
excerpt: Kubernetes RelicaSetController代码简析
mathjax: true
typora-root-url: ../
---

# ReplicaSet定义

explain一下

```shell
[root@rain-kubernetes-1 rain]# kubectl explain replicaset
KIND:     ReplicaSet
VERSION:  apps/v1

DESCRIPTION:
     ReplicaSet ensures that a specified number of pod replicas are running at
     any given time.

FIELDS:
   apiVersion   <string>
     APIVersion defines the versioned schema of this representation of an
     object. Servers should convert recognized schemas to the latest internal
     value, and may reject unrecognized values. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources

   kind <string>
     Kind is a string value representing the REST resource this object
     represents. Servers may infer this from the endpoint the client submits
     requests to. Cannot be updated. In CamelCase. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds

   metadata     <Object>
     If the Labels of a ReplicaSet are empty, they are defaulted to be the same
     as the Pod(s) that the ReplicaSet manages. Standard object's metadata. More
     info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata

   spec <Object>
     Spec defines the specification of the desired behavior of the ReplicaSet.
     More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status

   status       <Object>
     Status is the most recently observed status of the ReplicaSet. This data
     may be out of date by some window of time. Populated by the system.
     Read-only. More info:
     https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
```

我们可以顺着version: apps/v1，找到replicaset的结构定义：

*k8s.io/kubernetes/pkg/apis/apps/types.go* 

```go
// ReplicaSet ensures that a specified number of pod replicas are running at any given time.
type ReplicaSet struct {
	metav1.TypeMeta
	// +optional
	metav1.ObjectMeta

	// Spec defines the desired behavior of this ReplicaSet.
	// +optional
	Spec ReplicaSetSpec

	// Status is the current status of this ReplicaSet. This data may be
	// out of date by some window of time.
	// +optional
	Status ReplicaSetStatus
}

type TypeMeta struct {
	// Kind is a string value representing the REST resource this object represents.
	// Servers may infer this from the endpoint the client submits requests to.
	// Cannot be updated.
	// In CamelCase.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds
	// +optional
	Kind string `json:"kind,omitempty" protobuf:"bytes,1,opt,name=kind"`

	// APIVersion defines the versioned schema of this representation of an object.
	// Servers should convert recognized schemas to the latest internal value, and
	// may reject unrecognized values.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources
	// +optional
	APIVersion string `json:"apiVersion,omitempty" protobuf:"bytes,2,opt,name=apiVersion"`
}
```

ReplicaSet定义由四部分组成，跟explain出来的结果是对应的：

* TypeMeta（API元数据）

  * Kind
  * APIVersion

* ObjectMeta（对象元数据）

  * Name
  * GenerateName - 在Name没有提供的时候生成的
  * Namespace
  * SelfLink - 1.21将要被remove
  * UID
  * ResourceVersion - 系统填写的，代表对象内部的version
  * Generation - 代表期望状态的由系统生成的
  * CreationTimestamp
  * DeletionTimestamp
  * DeletionGracePeriodSeconds
  * Labels - key-value, 在selector的时候可以作为match的对象
  * Annotations - 非结构化的key-value，不可被查询，对象修改的时候也要保留
  * OwnerReferences - 依赖于这个对象的对象的list，如果list里所有的对象都被删除了，那么这个对象会被垃圾回收。如果这个对象是受一个controller控制的，那么在这个list里的一个entry会point到这个controller，controller filed会被设置为true，managing controller只能有一个
  * Finalizers
  * ClusterName
  * ManagedFields

* ReplicaSetSpec

  ```go
  // ReplicaSetSpec is the specification of a ReplicaSet.
  // As the internal representation of a ReplicaSet, it must have
  // a Template set.
  type ReplicaSetSpec struct {
  	// Replicas is the number of desired replicas.
  	Replicas int32
  
  	// Minimum number of seconds for which a newly created pod should be ready
  	// without any of its container crashing, for it to be considered available.
  	// Defaults to 0 (pod will be considered available as soon as it is ready)
  	// +optional
  	MinReadySeconds int32
  
  	// Selector is a label query over pods that should match the replica count.
  	// Must match in order to be controlled.
  	// If empty, defaulted to labels on pod template.
  	// More info: https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors
  	// +optional
  	Selector *metav1.LabelSelector
  
  	// Template is the object that describes the pod that will be created if
  	// insufficient replicas are detected.
  	// +optional
  	Template api.PodTemplateSpec
  }
  ```

* ReplicaSetStatus

  ```go
  // ReplicaSetStatus represents the current status of a ReplicaSet.
  type ReplicaSetStatus struct {
  	// Replicas is the number of actual replicas.
  	Replicas int32
  
  	// The number of pods that have labels matching the labels of the pod template of the replicaset.
  	// +optional
  	FullyLabeledReplicas int32
  
  	// The number of ready replicas for this replica set.
  	// +optional
  	ReadyReplicas int32
  
  	// The number of available replicas (ready for at least minReadySeconds) for this replica set.
  	// +optional
  	AvailableReplicas int32
  
  	// ObservedGeneration is the most recent generation observed by the controller.
  	// +optional
  	ObservedGeneration int64
  
  	// Represents the latest available observations of a replica set's current state.
  	// +optional
  	Conditions []ReplicaSetCondition
  }
  ```

*k8s.io/kubernetes/pkg/apis/apps/v1/register.go* &  *k8s.io/kubernetes/pkg/apis/apps/register.go* 

这里放置了一些全局变量，以及注册type到APIServer中的代码，apps这个包下面不只有ReplicaSet，还有Deployment等其他对象

```go
// Adds the list of known types to the given scheme.
func addKnownTypes(scheme *runtime.Scheme) error {
	// TODO this will get cleaned up with the scheme types are fixed
	scheme.AddKnownTypes(SchemeGroupVersion,
		&DaemonSet{},
		&DaemonSetList{},
		&Deployment{},
		&DeploymentList{},
		&DeploymentRollback{},
		&autoscaling.Scale{},
		&StatefulSet{},
		&StatefulSetList{},
		&ControllerRevision{},
		&ControllerRevisionList{},
		&ReplicaSet{},
		&ReplicaSetList{},
	)
	return nil
}
```

*k8s.io/kubernetes/pkg/apis/apps/v1/doc.go* & *k8s.io/kubernetes/pkg/apis/apps/doc.go* 这里是Golang的文档源码文件

```go
// +k8s:conversion-gen=k8s.io/kubernetes/pkg/apis/apps
// +k8s:conversion-gen-external-types=k8s.io/api/apps/v1
// +k8s:defaulter-gen=TypeMeta
// +k8s:defaulter-gen-input=../../../../vendor/k8s.io/api/apps/v1  

package v1 // import "k8s.io/kubernetes/pkg/apis/apps/v1"
```

```go
// +k8s:deepcopy-gen=package
// +k8s:deepcopy-gen=package 意思是，请为整个apps包里的所有类型定义自动生成DeepCopy方法

package apps // import "k8s.io/kubernetes/pkg/apis/apps"
```

doc.go里 +<tag_name>[=value] 格式的注释，这就是 Kubernetes 进行代码生成要用的 Annotation 风格的注释

*k8s.io/kubernetes/pkg/apis/apps/v1/defaults.go* 里定义了默认对象的生成方法

```go
func SetDefaults_ReplicaSet(obj *appsv1.ReplicaSet) {
   if obj.Spec.Replicas == nil {
      obj.Spec.Replicas = new(int32)
      *obj.Spec.Replicas = 1
   }
}
```

 比如ReplicaSet的Rplicas不设置，默认就是1

# Informer，Lister

每个对象都有对应的Informer，Lister，controller部分的实现是依赖client-go库实现的

ReplicaSet的Informer定义在 *k8s.io/client-go/informers/apps/v1/replicaset.go*

```go
// ReplicaSetInformer provides access to a shared informer and lister for
// ReplicaSets.
type ReplicaSetInformer interface {
	Informer() cache.SharedIndexInformer
	Lister() v1.ReplicaSetLister
}

// NewFilteredReplicaSetInformer constructs a new informer for ReplicaSet type.
// Always prefer using an informer factory to get a shared informer instead of getting an independent
// one. This reduces memory footprint and number of connections to the server.
func NewFilteredReplicaSetInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers, tweakListOptions internalinterfaces.TweakListOptionsFunc) cache.SharedIndexInformer {
	return cache.NewSharedIndexInformer(
		&cache.ListWatch{
			ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.AppsV1().ReplicaSets(namespace).List(options)
			},
			WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.AppsV1().ReplicaSets(namespace).Watch(options)
			},
		},
		&appsv1.ReplicaSet{},
		resyncPeriod,
		indexers,
	)
}
```

# controller

Replicaset controller是在controller-manager里的，controller-manager启动的时候会把所有的controller都start起来，业务逻辑都是在controller里的，也就是，怎样从当前状态，达到desired状态

```go
func startReplicaSetController(ctx ControllerContext) (http.Handler, bool, error) {
   if !ctx.AvailableResources[schema.GroupVersionResource{Group: "apps", Version: "v1", Resource: "replicasets"}] {
      return nil, false, nil
   }
   go replicaset.NewReplicaSetController(
      ctx.InformerFactory.Apps().V1().ReplicaSets(),
      ctx.InformerFactory.Core().V1().Pods(),
      ctx.ClientBuilder.ClientOrDie("replicaset-controller"),
      replicaset.BurstReplicas,
   ).Run(int(ctx.ComponentConfig.ReplicaSetController.ConcurrentRSSyncs), ctx.Stop)
   return nil, true, nil
}
```

```go
// NewReplicaSetController configures a replica set controller with the specified event recorder
func NewReplicaSetController(rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, kubeClient clientset.Interface, burstReplicas int) *ReplicaSetController {
	eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartLogging(klog.Infof)
	eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: kubeClient.CoreV1().Events("")})
	return NewBaseController(rsInformer, podInformer, kubeClient, burstReplicas,
		apps.SchemeGroupVersion.WithKind("ReplicaSet"),
		"replicaset_controller",
		"replicaset",
		controller.RealPodControl{
			KubeClient: kubeClient,
			Recorder:   eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "replicaset-controller"}),
		},
	)
}
```

```go
// NewBaseController is the implementation of NewReplicaSetController with additional injected
// parameters so that it can also serve as the implementation of NewReplicationController.
// replicaset controller定义了两个informer，rsinformer, podinformer，也就是说replicaset会关注rs和pod的变化
func NewBaseController(rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, kubeClient clientset.Interface, burstReplicas int,
   gvk schema.GroupVersionKind, metricOwnerName, queueName string, podControl controller.PodControlInterface) *ReplicaSetController {
   if kubeClient != nil && kubeClient.CoreV1().RESTClient().GetRateLimiter() != nil {
      ratelimiter.RegisterMetricAndTrackRateLimiterUsage(metricOwnerName, kubeClient.CoreV1().RESTClient().GetRateLimiter())
   }

   rsc := &ReplicaSetController{
      GroupVersionKind: gvk,
      kubeClient:       kubeClient,
      podControl:       podControl,
      burstReplicas:    burstReplicas,
      expectations:     controller.NewUIDTrackingControllerExpectations(controller.NewControllerExpectations()),
      queue:            workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), queueName),
   }

   rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
      AddFunc:    rsc.enqueueReplicaSet,
      UpdateFunc: rsc.updateRS,
      // This will enter the sync loop and no-op, because the replica set has been deleted from the store.
      // Note that deleting a replica set immediately after scaling it to 0 will not work. The recommended
      // way of achieving this is by performing a `stop` operation on the replica set.
      DeleteFunc: rsc.enqueueReplicaSet,
   })
   rsc.rsLister = rsInformer.Lister() //在开始SyncLoop之前，先要list所有的replicaset
   rsc.rsListerSynced = rsInformer.Informer().HasSynced

   podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
      AddFunc: rsc.addPod,
      // This invokes the ReplicaSet for every pod change, eg: host assignment. Though this might seem like
      // overkill the most frequent pod update is status, and the associated ReplicaSet will only list from
      // local storage, so it should be ok.
      UpdateFunc: rsc.updatePod,
      DeleteFunc: rsc.deletePod, // 添加add， update, delete
   })
   rsc.podLister = podInformer.Lister() //在开始SyncLoop之前，先要list所有的pod
   rsc.podListerSynced = podInformer.Informer().HasSynced

   rsc.syncHandler = rsc.syncReplicaSet

   return rsc
}
```

在Run里会启动控制循环:

* 等待 Informer 完成一次本地缓存的数据同步操作

* 通过 goroutine 启动一个（或者并发启动多个，根据ConcurrentRSSyncs的定义，决定启动多少worker）“无限循环”的任务，这无限循环里的任务，就是真正的业务逻辑。后面的处理其实就是红色部分的内容。

  ![image-20191124225004461](/../assets/images/image-20191124225004461.png)

```go
// worker runs a worker thread that just dequeues items, processes them, and marks them done.
// It enforces that the syncHandler is never invoked concurrently with the same key.
func (rsc *ReplicaSetController) worker() {
	for rsc.processNextWorkItem() {
	}
}

func (rsc *ReplicaSetController) processNextWorkItem() bool {
  key, quit := rsc.queue.Get()  //从工作队列里出队一个key(对象的key)，key是handler加到工作队列里的，可能是add, delete或者update
	if quit {
		return false
	}
	defer rsc.queue.Done(key)

	err := rsc.syncHandler(key.(string)) //rsc.syncHandler = rsc.syncReplicaSet
	if err == nil {
		rsc.queue.Forget(key)
		return true
	}

	utilruntime.HandleError(fmt.Errorf("Sync %q failed with %v", key, err))
	rsc.queue.AddRateLimited(key)

	return true
}
```

```go
// syncReplicaSet will sync the ReplicaSet with the given key if it has had its expectations fulfilled,
// meaning it did not expect to see any more of its pods created or deleted. This function is not meant to be
// invoked concurrently with the same key.
func (rsc *ReplicaSetController) syncReplicaSet(key string) error {

	startTime := time.Now()
	defer func() {
		klog.V(4).Infof("Finished syncing %v %q (%v)", rsc.Kind, key, time.Since(startTime))
	}()

	namespace, name, err := cache.SplitMetaNamespaceKey(key) //从key里解析出namespace和replicaset name
	if err != nil {
		return err
	}
	rs, err := rsc.rsLister.ReplicaSets(namespace).Get(name) //从Lister里查找对象，这个Lister是ReplicaSets的store, 通过shared informer传递到controller的，是descried的状态
	if errors.IsNotFound(err) {
		klog.V(4).Infof("%v %v has been deleted", rsc.Kind, key)
		rsc.expectations.DeleteExpectations(key) //在Lister里找不到，说明这个对象被删除了，从expectations删除对应的key
		return nil
	}
	if err != nil {
		return err
	}

	rsNeedsSync := rsc.expectations.SatisfiedExpectations(key) //判断是否需要sync，是否需要去比较期望值跟当前值是否一样
	selector, err := metav1.LabelSelectorAsSelector(rs.Spec.Selector)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("Error converting pod selector to selector: %v", err))
		return nil
	}

	// list all pods to include the pods that don't match the rs`s selector
	// anymore but has the stale controller ref.
	// TODO: Do the List and Filter in a single pass, or use an index.
	allPods, err := rsc.podLister.Pods(rs.Namespace).List(labels.Everything()) //list namespace下所有的pod
	if err != nil {
		return err
	}
	// Ignore inactive pods. 忽略没有active的pods
	filteredPods := controller.FilterActivePods(allPods)

	// NOTE: filteredPods are pointing to objects from cache - if you need to
	// modify them, you need to copy it first. 过滤出所有属于这个replicaset的pod放到filteredPods里
	filteredPods, err = rsc.claimPods(rs, selector, filteredPods)
	if err != nil {
		return err
	}

	var manageReplicasErr error
	if rsNeedsSync && rs.DeletionTimestamp == nil { //需要作对比，且replicaset并没有被删除
		manageReplicasErr = rsc.manageReplicas(filteredPods, rs) //检查是否要增加或者删除pod以满足期望值
	}
	rs = rs.DeepCopy()
	newStatus := calculateStatus(rs, filteredPods, manageReplicasErr) //更新状态

	// Always updates status as pods come up or die.
	updatedRS, err := updateReplicaSetStatus(rsc.kubeClient.AppsV1().ReplicaSets(rs.Namespace), rs, newStatus)
	if err != nil {
		// Multiple things could lead to this update failing. Requeuing the replica set ensures
		// Returning an error causes a requeue without forcing a hotloop
		return err
	}
	// Resync the ReplicaSet after MinReadySeconds as a last line of defense to guard against clock-skew.
	if manageReplicasErr == nil && updatedRS.Spec.MinReadySeconds > 0 &&
		updatedRS.Status.ReadyReplicas == *(updatedRS.Spec.Replicas) &&
		updatedRS.Status.AvailableReplicas != *(updatedRS.Spec.Replicas) {
		rsc.enqueueReplicaSetAfter(updatedRS, time.Duration(updatedRS.Spec.MinReadySeconds)*time.Second)
	}
	return manageReplicasErr
}
```

expectations是用来记录需要删除的pod的uid，使得他们不被重复计算

```go
	// A TTLCache of pod creates/deletes each rc expects to see.
	expectations *controller.UIDTrackingControllerExpectations
	
// UIDTrackingControllerExpectations tracks the UID of the pods it deletes.
// This cache is needed over plain old expectations to safely handle graceful
// deletion. The desired behavior is to treat an update that sets the
// DeletionTimestamp on an object as a delete. To do so consistently, one needs
// to remember the expected deletes so they aren't double counted.
// TODO: Track creates as well (#22599)
type UIDTrackingControllerExpectations struct {
	ControllerExpectationsInterface
	// TODO: There is a much nicer way to do this that involves a single store,
	// a lock per entry, and a ControlleeExpectationsInterface type.
	uidStoreLock sync.Mutex
	// Store used for the UIDs associated with any expectation tracked via the
	// ControllerExpectationsInterface.
	uidStore cache.Store
}
```

controller的syncLoop其实充当的是消费者，从workqueue不断地取key进行处理，而图左侧Informer左侧，则是生产者，不断往workqueue里塞数据，后面会再分析。