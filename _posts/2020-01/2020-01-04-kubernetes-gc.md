---
layout: post
title:  "Kubernetes垃圾回收机制"
date:   2020-01-04 20:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Kubelet
excerpt: Kubernetes垃圾回收机制
mathjax: true
typora-root-url: ../
---

# Kubernetes GC

kubernetes GC是由kubelet来完成的，kubelet除了根据 apiserver 分配的 pod 创建容器之外，它还有其他很多事情要做，其中之一就是 GC（Garbage Collect）。

在运行一段时候之后，节点上会下载很多镜像，也会有很多因为各种原因退出的容器。为了保证节点能够正常运行，kubelet 要防止镜像太多占满磁盘空间，也要防止退出的容器太多导致系统运行缓慢或者出现错误。

GC 的工作不需要手动干预，kubelet 会周期性去执行，不过在启动 kubelet 进程的时候可以通过参数控制 GC 的策略。

# 容器GC

退出的容器会继续占用系统资源，比如还会在文件系统存储很多数据、docker 应用也要占用 CPU 和内存去维护这些容器。docker 本身并不会自动删除已经退出的容器，因此 kubelet 就负起了这个责任。kubelet 容器的回收是为了删除已经退出的容器以节省节点的空间，提升性能。

和容器 GC 有关的可以配置的 kubelet 启动参数包括：

- `minimum-container-ttl-duration`：container 结束多长时间之后才能够被回收，默认是一分钟
- `maximum-dead-containers-per-container`：每个 container 最终可以保存多少个已经结束的容器，默认是 1，设置为负数表示不做限制
- `maximum-dead-containers`：节点上最多能保留多少个结束的容器，默认是 -1，表示不做限制

默认情况下，kubelet 会自动每分钟去做容器 GC，容器退出一分钟之后就可以被删除，而且每个容器做多只会保留一个已经退出的历史容器。

# 镜像GC

镜像主要占用磁盘空间，虽然 docker 使用镜像分层可以让多个镜像共享存储，但是长时间运行的节点如果下载了很多镜像也会导致占用的存储空间过多。如果镜像导致磁盘被占满，会造成应用无法正常工作。docker 默认也不会做镜像清理，镜像一旦下载就会永远留在本地，除非被手动删除。

镜像的清理和容器不同，是以占用的空间作为标准的，用户可以配置当镜像占据多大比例的存储空间时才进行清理。清理的时候会优先清理最久没有被使用的镜像，镜像被 pull 下来或者被容器使用都会更新它的最近使用时间。

启动 kubelet 的时候，可以配置这些参数控制镜像清理的策略：

- `image-gc-high-threshold`：磁盘使用率的上限，当达到这一使用率的时候会触发镜像清理。默认值为 90%
- `image-gc-low-threshold`：磁盘使用率的下限，每次清理直到使用率低于这个值或者没有可以清理的镜像了才会停止。默认值为 80%
- `minimum-image-ttl-duration`：镜像最少这么久没有被使用才会被清理，可以使用 `h`（小时）、`m`（分钟）、`s`（秒）和 `ms`（毫秒）时间单位进行配置，默认是 `2m`(两分钟)

默认情况下，当镜像占满所在盘 90% 容量的时候，kubelet 就会进行清理，一直到镜像占用率低于 80% 为止。

# GC代码分析

在启动kubelet的时候，会StartGarbageCollection，这个就是GC。容器 GC 和镜像 GC 分别是在独立的 goroutine 中执行的

```go
// StartGarbageCollection starts garbage collection threads.
func (kl *Kubelet) StartGarbageCollection() {
   loggedContainerGCFailure := false
   go wait.Until(func() {
      //容器GC
      if err := kl.containerGC.GarbageCollect(); err != nil {
         klog.Errorf("Container garbage collection failed: %v", err)
         kl.recorder.Eventf(kl.nodeRef, v1.EventTypeWarning, events.ContainerGCFailed, err.Error())
         loggedContainerGCFailure = true
      } else {
         var vLevel klog.Level = 4
         if loggedContainerGCFailure {
            vLevel = 1
            loggedContainerGCFailure = false
         }

         klog.V(vLevel).Infof("Container garbage collection succeeded")
      }
   }, ContainerGCPeriod, wait.NeverStop)

   // when the high threshold is set to 100, stub the image GC manager
   if kl.kubeletConfiguration.ImageGCHighThresholdPercent == 100 {
      klog.V(2).Infof("ImageGCHighThresholdPercent is set 100, Disable image GC")
      return
   }

   prevImageGCFailed := false
   go wait.Until(func() {
      //镜像GC
      if err := kl.imageManager.GarbageCollect(); err != nil {
         if prevImageGCFailed {
            klog.Errorf("Image garbage collection failed multiple times in a row: %v", err)
            // Only create an event for repeated failures
            kl.recorder.Eventf(kl.nodeRef, v1.EventTypeWarning, events.ImageGCFailed, err.Error())
         } else {
            klog.Errorf("Image garbage collection failed once. Stats initialization may not have completed yet: %v", err)
         }
         prevImageGCFailed = true
      } else {
         var vLevel klog.Level = 4
         if prevImageGCFailed {
            vLevel = 1
            prevImageGCFailed = false
         }

         klog.V(vLevel).Infof("Image garbage collection succeeded")
      }
   }, ImageGCPeriod, wait.NeverStop)
}
```

## 容器GC

```go
// ContainerGCPeriod is the period for performing container garbage collection.
ContainerGCPeriod = time.Minute
```

1分钟会执行一次容器GC

```go
func (cgc *realContainerGC) GarbageCollect() error {
   return cgc.runtime.GarbageCollect(cgc.policy, cgc.sourcesReadyProvider.AllReady(), false)
}

	// GarbageCollect removes dead containers using the specified container gc policy
	// If allSourcesReady is not true, it means that kubelet doesn't have the
	// complete list of pods from all avialble sources (e.g., apiserver, http,
	// file). In this case, garbage collector should refrain itself from aggressive
	// behavior such as removing all containers of unrecognized pods (yet).
	// If evictNonDeletedPods is set to true, containers and sandboxes belonging to pods
	// that are terminated, but not deleted will be evicted.  Otherwise, only deleted pods will be GC'd.
	// TODO: Revisit this method and make it cleaner.
  // ContainerGCPolicy是容器gc的策略，如果allSourcesReady是false，说明kubelet还没能去能所有pod的列表，那么这个时候不应该进行gc，如果evictNonDeletedPods是true，归属于pod的terminated的container&sandbox会被evicted，否则，只有删除的pod还会被gc
	GarbageCollect(gcPolicy ContainerGCPolicy, allSourcesReady bool, evictNonDeletedPods bool) error

// GarbageCollect removes dead containers using the specified container gc policy.
func (m *kubeGenericRuntimeManager) GarbageCollect(gcPolicy kubecontainer.ContainerGCPolicy, allSourcesReady bool, evictNonDeletedPods bool) error {
	return m.containerGC.GarbageCollect(gcPolicy, allSourcesReady, evictNonDeletedPods)
}

// GarbageCollect removes dead containers using the specified container gc policy.
// Note that gc policy is not applied to sandboxes. Sandboxes are only removed when they are
// not ready and containing no containers.
//
// GarbageCollect consists of the following steps:
// * gets evictable containers which are not active and created more than gcPolicy.MinAge ago.
// * removes oldest dead containers for each pod by enforcing gcPolicy.MaxPerPodContainer.
// * removes oldest dead containers by enforcing gcPolicy.MaxContainers.
// * gets evictable sandboxes which are not ready and contains no containers.
// * removes evictable sandboxes.
func (cgc *containerGC) GarbageCollect(gcPolicy kubecontainer.ContainerGCPolicy, allSourcesReady bool, evictTerminatedPods bool) error {
	errors := []error{}
	// Remove evictable containers
  // 找出不在running而且创建时间大于minimum-container-ttl-duration的容器，如果pod是deleted或者terminated，就删除
	if err := cgc.evictContainers(gcPolicy, allSourcesReady, evictTerminatedPods); err != nil {
		errors = append(errors, err)
	}

	// Remove sandboxes with zero containers
  // 删除所有avictable的sandbox(not in ready state，contains no containers, belong to a non-existent pod or is not the most recently created sandbox for the pod)
	if err := cgc.evictSandboxes(evictTerminatedPods); err != nil {
		errors = append(errors, err)
	}

	// Remove pod sandbox log directory，evicts all evictable pod logs directories
	if err := cgc.evictPodLogsDirectories(allSourcesReady); err != nil {
		errors = append(errors, err)
	}
	return utilerrors.NewAggregate(errors)
}
```

## 镜像GC

```go
// ImageGCPeriod is the period for performing image garbage collection.
ImageGCPeriod = 5 * time.Minute
```

5分钟执行一次镜像GC

```go
func (im *realImageGCManager) GarbageCollect() error {
   // Get disk usage on disk holding images.
   fsStats, err := im.statsProvider.ImageFsStats()
   if err != nil {
      return err
   }

   var capacity, available int64
   if fsStats.CapacityBytes != nil {
      capacity = int64(*fsStats.CapacityBytes)
   }
   if fsStats.AvailableBytes != nil {
      available = int64(*fsStats.AvailableBytes)
   }

   if available > capacity {
      klog.Warningf("available %d is larger than capacity %d", available, capacity)
      available = capacity
   }

   // Check valid capacity.
   if capacity == 0 {
      err := goerrors.New("invalid capacity 0 on image filesystem")
      im.recorder.Eventf(im.nodeRef, v1.EventTypeWarning, events.InvalidDiskCapacity, err.Error())
      return err
   }

   // If over the max threshold, free enough to place us at the lower threshold.
   // 如果镜像的磁盘使用率达到了设定的最高阈值，就进行清理工作
   usagePercent := 100 - int(available*100/capacity)
   if usagePercent >= im.policy.HighThresholdPercent {
      amountToFree := capacity*int64(100-im.policy.LowThresholdPercent)/100 - available
      klog.Infof("[imageGCManager]: Disk usage on image filesystem is at %d%% which is over the high threshold (%d%%). Trying to free %d bytes down to the low threshold (%d%%).", usagePercent, im.policy.HighThresholdPercent, amountToFree, im.policy.LowThresholdPercent)
      freed, err := im.freeSpace(amountToFree, time.Now())
      if err != nil {
         return err
      }

      if freed < amountToFree {
         err := fmt.Errorf("failed to garbage collect required amount of images. Wanted to free %d bytes, but freed %d bytes", amountToFree, freed)
         im.recorder.Eventf(im.nodeRef, v1.EventTypeWarning, events.FreeDiskSpaceFailed, err.Error())
         return err
      }
   }

   return nil
}
```

image gc的时候需要知道磁盘的情况，这里有个statsProvider，对于runtime是docker而且系统是linux的情况，使用cadvisor，否则用CRI的接口。docker uses the built-in cadvisor to gather such metrics on Linux for historical reasons.