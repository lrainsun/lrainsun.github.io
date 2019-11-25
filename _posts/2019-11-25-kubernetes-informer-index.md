---
layout: post
title:  "Kubernetes Informer Indexer & Local Store代码简析"
date:   2019-11-25 21:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Controller
excerpt: Kubernetes Informer Indexer & Local Store代码简析
mathjax: true
typora-root-url: ../
---

# Indexer & Local Store基本概念

![image-20191125085228713](/assets/images/image-20191125085228713.png)

依然是这张图，Indexer & Local Store的位置在红色框起来的部分。

Informer会从Delta FIFO Queue 里读取（Pop）增量：

* 每拿到一个增量，Informer 就会判断这个增量里的事件类型，然后创建或者更新本地对象的缓存
* 这个缓存，在 Kubernetes 里一般被叫作 Store。

如果事件类型是 Added（添加对象），那么 Informer 就会通过一个叫作 Indexer 的库把这个增量里的 API 对象保存在本地缓存中，并为它创建索引。相反地，如果增量的事件类型是 Deleted（删除对象），那么 Informer 就会从本地缓存中删除这个对象。

而这个Store里的内容最终会被上次学习的controller的SyncLoop作为期望状态用来作比较。

# Indexer & Local Store代码实现

从实现上来说，Indexer实际上是包含了Store

Store:

```go
// Store is a generic object storage interface. Reflector knows how to watch a server
// and update a store. A generic store is provided, which allows Reflector to be used
// as a local caching system, and an LRU store, which allows Reflector to work like a
// queue of items yet to be processed.
//
// Store makes no assumptions about stored object identity; it is the responsibility
// of a Store implementation to provide a mechanism to correctly key objects and to
// define the contract for obtaining objects by some arbitrary key type.
type Store interface {
	Add(obj interface{}) error
	Update(obj interface{}) error
	Delete(obj interface{}) error
	List() []interface{}
	ListKeys() []string
	Get(obj interface{}) (item interface{}, exists bool, err error)
	GetByKey(key string) (item interface{}, exists bool, err error)

	// Replace will delete the contents of the store, using instead the
	// given list. Store takes ownership of the list, you should not reference
	// it after calling this function.
	Replace([]interface{}, string) error
	Resync() error
}
```

Indexer，在store的基础上，增加了几个index的方法

```go
// Indexer is a storage interface that lets you list objects using multiple indexing functions.
// There are three kinds of strings here.
// One is a storage key, as defined in the Store interface.
// Another kind is a name of an index.
// The third kind of string is an "indexed value", which is produced by an
// IndexFunc and can be a field value or any other string computed from the object.
type Indexer interface {
	Store
	// Index returns the stored objects whose set of indexed values
	// intersects the set of indexed values of the given object, for
	// the named index
	Index(indexName string, obj interface{}) ([]interface{}, error)
	// IndexKeys returns the storage keys of the stored objects whose
	// set of indexed values for the named index includes the given
	// indexed value
	IndexKeys(indexName, indexedValue string) ([]string, error)
	// ListIndexFuncValues returns all the indexed values of the given index
	ListIndexFuncValues(indexName string) []string
	// ByIndex returns the stored objects whose set of indexed values
	// for the named index includes the given indexed value
	ByIndex(indexName, indexedValue string) ([]interface{}, error)
	// GetIndexer return the indexers
	GetIndexers() Indexers

	// AddIndexers adds more indexers to this store.  If you call this after you already have data
	// in the store, the results are undefined.
	AddIndexers(newIndexers Indexers) error
}
```

找一下Indexer的实现：

```go
// cache responsibilities are limited to:
// 1. Computing keys for objects via keyFunc
//  2. Invoking methods of a ThreadSafeStorage interface
type cache struct {
   // cacheStorage bears the burden of thread safety for the cache. store的threadsaft的实现
   cacheStorage ThreadSafeStore 
   // keyFunc is used to make the key for objects stored in and retrieved from items, and
   // should be deterministic. 从对象获取它的Key 
   keyFunc KeyFunc
}

// Add inserts an item into the cache.
func (c *cache) Add(obj interface{}) error {
	key, err := c.keyFunc(obj)
	if err != nil {
		return KeyError{obj, err}
	}
	c.cacheStorage.Add(key, obj)
	return nil
}

// GetIndexers returns the indexers of cache
func (c *cache) GetIndexers() Indexers {
	return c.cacheStorage.GetIndexers()
}
```

```go
// ThreadSafeStore is an interface that allows concurrent access to a storage backend.
// TL;DR caveats: you must not modify anything returned by Get or List as it will break
// the indexing feature in addition to not being thread safe.
//
// The guarantees of thread safety provided by List/Get are only valid if the caller
// treats returned items as read-only. For example, a pointer inserted in the store
// through `Add` will be returned as is by `Get`. Multiple clients might invoke `Get`
// on the same key and modify the pointer in a non-thread-safe way. Also note that
// modifying objects stored by the indexers (if any) will *not* automatically lead
// to a re-index. So it's not a good idea to directly modify the objects returned by
// Get/List, in general.
type ThreadSafeStore interface {
	Add(key string, obj interface{})
	Update(key string, obj interface{})
	Delete(key string)
	Get(key string) (item interface{}, exists bool)
	List() []interface{}
	ListKeys() []string
	Replace(map[string]interface{}, string)
	Index(indexName string, obj interface{}) ([]interface{}, error)
	IndexKeys(indexName, indexKey string) ([]string, error)
	ListIndexFuncValues(name string) []string
	ByIndex(indexName, indexKey string) ([]interface{}, error)
	GetIndexers() Indexers

	// AddIndexers adds more indexers to this store.  If you call this after you already have data
	// in the store, the results are undefined.
	AddIndexers(newIndexers Indexers) error
	Resync() error
}
```

threadSafeMap是ThreadSafeStore的实现

```go
// threadSafeMap implements ThreadSafeStore
type threadSafeMap struct {
   lock  sync.RWMutex
   items map[string]interface{}

   // indexers maps a name to an IndexFunc
   indexers Indexers
   // indices maps a name to an Index
   indices Indices
}
```

之前看的replicaset初始化informer的代码，重点看一下indexer的初始化

```go
// NewSharedIndexInformer creates a new instance for the listwatcher.
func NewSharedIndexInformer(lw ListerWatcher, objType runtime.Object, defaultEventHandlerResyncPeriod time.Duration, indexers Indexers) SharedIndexInformer {
	realClock := &clock.RealClock{}
	sharedIndexInformer := &sharedIndexInformer{
		processor:                       &sharedProcessor{clock: realClock},
		indexer:                         NewIndexer(DeletionHandlingMetaNamespaceKeyFunc, indexers),
		listerWatcher:                   lw,
		objectType:                      objType,
		resyncCheckPeriod:               defaultEventHandlerResyncPeriod,
		defaultEventHandlerResyncPeriod: defaultEventHandlerResyncPeriod,
		cacheMutationDetector:           NewCacheMutationDetector(fmt.Sprintf("%T", objType)),
		clock:                           realClock,
	}
	return sharedIndexInformer
}

// NewIndexer returns an Indexer implemented simply with a map and a lock.
func NewIndexer(keyFunc KeyFunc, indexers Indexers) Indexer {
	return &cache{
		cacheStorage: NewThreadSafeStore(indexers, Indices{}),
		keyFunc:      keyFunc,
	}
}
```

***DeletionHandlingMetaNamespaceKeyFunc*** —— keyFunc，根据obj获取key的方法

***indexers*** —— Indexers，name到IndexFunc的Map，IndexFunc计算obj的一组indexed value

# syncLoop

replicaset的sysnLoop里会调用Lister来获取期望状态，昨天学习的

```go
	rs, err := rsc.rsLister.ReplicaSets(namespace).Get(name) //从Lister里查找对象，这个Lister是ReplicaSets的store, 通过shared informer传递到controller的，是descried的状态
```

仔细看一下这里的rsc.rsLister，其实是从cache.ListAll

```go
	// A store of ReplicaSets, populated by the shared informer passed to NewReplicaSetController
	rsLister appslisters.ReplicaSetLister

// ReplicaSetLister helps list ReplicaSets.
type ReplicaSetLister interface {
	// List lists all ReplicaSets in the indexer.
	List(selector labels.Selector) (ret []*v1.ReplicaSet, err error)
	// ReplicaSets returns an object that can list and get ReplicaSets.
	ReplicaSets(namespace string) ReplicaSetNamespaceLister
	ReplicaSetListerExpansion
}

// List lists all ReplicaSets in the indexer.
func (s *replicaSetLister) List(selector labels.Selector) (ret []*v1.ReplicaSet, err error) {
	err = cache.ListAll(s.indexer, selector, func(m interface{}) {
		ret = append(ret, m.(*v1.ReplicaSet))
	})
	return ret, err
}
```

cache的实现在*k8s.io/client-go/tools/cache/lister.go*，会调用store.List，我们传入的store是s.indexer。最终会调用cacheStorage.List()，接口是ThreadSafeStore，而实现的正是threadSafeMap

```go
// ListAll calls appendFn with each value retrieved from store which matches the selector.
func ListAll(store Store, selector labels.Selector, appendFn AppendFunc) error {
  for _, m := range store.List() {
  }
}

// List returns a list of all the items.
// List is completely threadsafe as long as you treat all items as immutable.
func (c *cache) List() []interface{} {
	return c.cacheStorage.List()
}
```

