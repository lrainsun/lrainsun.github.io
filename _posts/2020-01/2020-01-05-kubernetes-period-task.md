---
layout: post
title:  "Kubernetes定时轮询"
date:   2020-01-05 23:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Controller
excerpt: Kubernetes定时轮询
mathjax: true
typora-root-url: ../
---

# kubernetes中的定时轮询

昨天在学习gc的时候，有这样一段代码，不管是容器gc还是镜像gc，都是一个永远不会结束的任务，并且定期会执行

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

这里用了`k8s.io/apimachinery/pkg/util/wait`里的方法Until，每隔period的时间执行一次f函数，直到stopCh被close掉

```go
// Until loops until stop channel is closed, running f every period.
//
// Until is syntactic sugar on top of JitterUntil with zero jitter factor and
// with sliding = true (which means the timer for period starts after the f
// completes).
func Until(f func(), period time.Duration, stopCh <-chan struct{}) {
   JitterUntil(f, period, 0.0, true, stopCh)
}
```

而传进去的第三个参数是NeverStop，就使得可以一直运行下去

```go
// NeverStop may be passed to Until to make it never stop.
var NeverStop <-chan struct{} = make(chan struct{})
```

其实还有一个方法，实现的功能是一样的

```go
// Forever calls f every period for ever.
//
// Forever is syntactic sugar on top of Until.
func Forever(f func(), period time.Duration) {
   Until(f, period, NeverStop)
}
```