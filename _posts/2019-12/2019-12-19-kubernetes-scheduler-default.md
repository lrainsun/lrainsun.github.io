---
layout: post
title:  "Kubernetes调度策略解析"
date:   2019-12-19 22:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Scheduler
excerpt: Kubernetes调度策略解析
mathjax: true
typora-root-url: ../
---

昨天学习了scheduler有两个阶段，Predicates&Priorities

# Predicates

按照调度策略，从当前集群的所有节点中，“过滤”出一系列符合条件的节点。这些节点，都是可以运行待调度 Pod 的宿主机。

## GeneralPredicates

最基础的调度策略

### PodFitsResources

宿主机的 CPU 和内存资源等是否够用，PodFitsResources 检查的只是 Pod 的 requests 字段

PodFitsResources checks if a node has sufficient resources, such as cpu, memory, gpu, opaque int resources etc to run a pod. 

* 检查node allowed pod number是否满足需要

```go
allowedPodNumber := nodeInfo.AllowedPodNumber()
if len(nodeInfo.Pods())+1 > allowedPodNumber {
   predicateFails = append(predicateFails, NewInsufficientResourceError(v1.ResourcePods, 1, int64(len(nodeInfo.Pods())), int64(allowedPodNumber)))
}
```

* 检查node resource是否满足需要

```go
allocatable := nodeInfo.AllocatableResource()
if allocatable.MilliCPU < podRequest.MilliCPU+nodeInfo.RequestedResource().MilliCPU {
   predicateFails = append(predicateFails, NewInsufficientResourceError(v1.ResourceCPU, podRequest.MilliCPU, nodeInfo.RequestedResource().MilliCPU, allocatable.MilliCPU))
}
if allocatable.Memory < podRequest.Memory+nodeInfo.RequestedResource().Memory {
   predicateFails = append(predicateFails, NewInsufficientResourceError(v1.ResourceMemory, podRequest.Memory, nodeInfo.RequestedResource().Memory, allocatable.Memory))
}
if allocatable.EphemeralStorage < podRequest.EphemeralStorage+nodeInfo.RequestedResource().EphemeralStorage {
   predicateFails = append(predicateFails, NewInsufficientResourceError(v1.ResourceEphemeralStorage, podRequest.EphemeralStorage, nodeInfo.RequestedResource().EphemeralStorage, allocatable.EphemeralStorage))
}
```

* 检查是否有额外的资源请求

```go
for rName, rQuant := range podRequest.ScalarResources {
   if v1helper.IsExtendedResourceName(rName) {
      // If this resource is one of the extended resources that should be
      // ignored, we will skip checking it.
      if ignoredExtendedResources.Has(string(rName)) {
         continue
      }
   }
   if allocatable.ScalarResources[rName] < rQuant+nodeInfo.RequestedResource().ScalarResources[rName] {
      predicateFails = append(predicateFails, NewInsufficientResourceError(rName, podRequest.ScalarResources[rName], nodeInfo.RequestedResource().ScalarResources[rName], allocatable.ScalarResources[rName]))
   }
}
```

### PodFitsHost

PodFitsHost 检查的是，宿主机的名字是否跟 Pod 的 spec.nodeName 一致

PodFitsHost checks if a pod spec node name matches the current node.

```go
node := nodeInfo.Node()
if node == nil {
   return false, nil, fmt.Errorf("node not found")
}
if pod.Spec.NodeName == node.Name {
   return true, nil, nil
}
```

### PodFitsHostPorts

PodFitsHostPorts 检查的是，Pod 申请的宿主机端口（spec.nodePort）是不是跟已经被使用
的端口有冲突

PodFitsHostPorts checks if a node has free ports for the requested pod ports.

```go
if predicateMeta, ok := meta.(*predicateMetadata); ok && predicateMeta.podFitsHostPortsMetadata != nil {
   wantPorts = predicateMeta.podFitsHostPortsMetadata.podPorts
} else {
   // We couldn't parse metadata - fallback to computing it.
   wantPorts = schedutil.GetContainerPorts(pod)
}
if len(wantPorts) == 0 {
   return true, nil, nil
}

existingPorts := nodeInfo.UsedPorts()

// try to see whether existingPorts and  wantPorts will conflict or not
if portsConflict(existingPorts, wantPorts) {
   return false, []PredicateFailureReason{ErrPodNotFitsHostPorts}, nil
}
```

### PodMatchNodeSelector

PodMatchNodeSelector 检查的是，Pod 的 nodeSelector 或者 nodeAffinity 指定的节点，是否与待考察节点匹配等

PodMatchNodeSelector checks if a pod node selector matches the node label

```go
if PodMatchesNodeSelectorAndAffinityTerms(pod, node) {
   return true, nil, nil
}
```

GeneralPredicates，是 Kubernetes 考察一个 Pod 能不能运行在一个 Node 上最基本的过滤条件。所以，GeneralPredicates 也会被其他组件（比如kubelet）直接调用。

kubelet 在启动 Pod 前，会执行一个 Admit 操作来进行二次确认。这里二次确认的规则，就是执行一遍 GeneralPredicates。In func (w *predicateAdmitHandler) Admit(attrs *PodAdmitAttributes) PodAdmitResult

```go
fit, reasons, err := predicates.GeneralPredicates(podWithoutMissingExtendedResources, nil, nodeInfo)
if err != nil {
   message := fmt.Sprintf("GeneralPredicates failed due to %v, which is unexpected.", err)
   klog.Warningf("Failed to admit pod %v - %s", format.Pod(admitPod), message)
   return PodAdmitResult{
      Admit:   fit,
      Reason:  "UnexpectedAdmissionError",
      Message: message,
   }
}
```

## 与Volume相关的过滤规则

这一组过滤规则，负责的是跟容器持久化 Volume 相关的调度策略。

### NoDiskConflict

NoDiskConflict 检查的条件，是多个 Pod 声明挂载的持久化 Volume 是否有冲突。比如，AWS EBS 类型的 Volume，是不允许被两个 Pod 同时使用的。所以，当一个名叫 A 的EBS Volume 已经被挂载在了某个节点上时，另一个同样声明使用这个 A Volume 的 Pod，就不能被调度到这个节点上了。

NoDiskConflict evaluates if a pod can fit due to the volumes it requests, and those that are already mounted. If there is already a volume mounted on that node, another pod that uses the same volume can't be scheduled there.

```go
for _, v := range pod.Spec.Volumes {
   for _, ev := range nodeInfo.Pods() {
      if isVolumeConflict(v, ev) {
         return false, []PredicateFailureReason{ErrDiskConflict}, nil
      }
   }
}
```

### VolumeZonePredicate

VolumeZonePredicate，则是检查持久化 Volume 的 Zone（高可用域）标签，是否与待考察节点的 Zone 标签相匹配。

```go
volumeVSet, err := volumehelpers.LabelZonesToSet(v)
if err != nil {
   klog.Warningf("Failed to parse label for %q: %q. Ignoring the label. err=%v. ", k, v, err)
   continue
}

if !volumeVSet.Has(nodeV) {
   klog.V(10).Infof("Won't schedule pod %q onto node %q due to volume %q (mismatch on %q)", pod.Name, node.Name, pvName, k)
   return false, []PredicateFailureReason{ErrVolumeZoneConflict}, nil
}
```

### VolumeBindingPredicate

检查的该 Pod 对应的PV 的 nodeAffinity 字段，是否跟某个节点的标签相匹配。

```go
if !boundSatisfied {
   klog.V(5).Infof("Bound PVs not satisfied for pod %v/%v, node %q", pod.Namespace, pod.Name, node.Name)
   failReasons = append(failReasons, ErrVolumeNodeConflict)
}

if !unboundSatisfied {
   klog.V(5).Infof("Couldn't find matching PVs for pod %v/%v, node %q", pod.Namespace, pod.Name, node.Name)
   failReasons = append(failReasons, ErrVolumeBindConflict)
}
```

Local Persistent Volume（本地持久化卷），必须使用 nodeAffinity 来跟某个具体的节点绑定。这其实也就意味着，在 Predicates 阶段，Kubernetes 就必须能够根据 Pod 的Volume 属性来进行调度。
此外，如果该 Pod 的 PVC 还没有跟具体的 PV 绑定的话，调度器还要负责检查所有待绑定PV，当有可用的 PV 存在并且该 PV 的 nodeAffinity 与待考察节点一致时，这条规则才会返回“成功”。

## 与宿主机相关的过滤规则

主要考察待调度 Pod 是否满足 Node 本身的某些条件。

### PodToleratesNodeTaints

负责检查的就是我们前面经常用到的 Node 的“污点”机制。只有当 Pod 的 Toleration 字段与 Node 的 Taint 字段能够匹配的时候，这个 Pod 才能被调度到该节点上。

```go
return podToleratesNodeTaints(pod, nodeInfo, func(t *v1.Taint) bool {
   // PodToleratesNodeTaints is only interested in NoSchedule and NoExecute taints.
   return t.Effect == v1.TaintEffectNoSchedule || t.Effect == v1.TaintEffectNoExecute
})
```

## Pod相关的过滤规则

这一组规则，跟 GeneralPredicates 大多数是重合的。而比较特殊的，是PodAffinityPredicate。这个规则的作用，是检查待调度 Pod 与 Node 上的已有 Pod 之间的亲密（affinity）和反亲密（anti-affinity）关系。

InterPodAffinityMatches checks if a pod can be scheduled on the specified node with pod affinity/anti-affinity configuration. First return value indicates whether a pod can be scheduled on the specified node while the second return value indicates the predicate failure reasons if the pod cannot be scheduled on the specified node.

```go
if failedPredicates, error := c.satisfiesExistingPodsAntiAffinity(pod, meta, nodeInfo); failedPredicates != nil {
   failedPredicates := append([]PredicateFailureReason{ErrPodAffinityNotMatch}, failedPredicates)
   return false, failedPredicates, error
}

// Now check if <pod> requirements will be satisfied on this node.
affinity := pod.Spec.Affinity
if affinity == nil || (affinity.PodAffinity == nil && affinity.PodAntiAffinity == nil) {
   return true, nil, nil
}
if failedPredicates, error := c.satisfiesPodsAffinityAntiAffinity(pod, meta, nodeInfo, affinity); failedPredicates != nil {
   failedPredicates := append([]PredicateFailureReason{ErrPodAffinityNotMatch}, failedPredicates)
   return false, failedPredicates, error
}
```

# Priorities

在 Predicates 阶段完成了节点的“过滤”之后，Priorities 阶段的工作就是为这些节点打分。这里打分的范围是 0-10 分，得分最高的节点就是最后被 Pod 绑定的最佳节点。

### LeastRequestedPriority

Priorities 里最常用到的一个打分规则，选择空闲资源（CPU 和 Memory）最多的宿主机

LeastRequestedPriorityMap is a priority function that favors nodes with fewer requested resources.
It calculates the percentage of memory and CPU requested by pods scheduled on the node, and
prioritizes based on the minimum of the average of the fraction of requested to capacity.
Details:
(cpu((capacity-sum(requested))*10/capacity) + memory((capacity-sum(requested))*10/capacity))/2

### BalancedResourceAllocation

每种资源的 Fraction 的定义是 ：Pod 请求的资源 / 节点上的可用资源。而 variance 算法的作用，则是计算每两种资源 Fraction 之间的“距离”。而最后选择的，则是资源 Fraction差距最小的节点。

BalancedResourceAllocationMap favors nodes with balanced resource usage rate. BalancedResourceAllocationMap should **NOT** be used alone, and **MUST** be used together with LeastRequestedPriority. It calculates the difference between the cpu and memory fraction of capacity, and prioritizes the host based on how close the two metrics are to each other.
Detail: score = 10 - variance(cpuFraction,memoryFraction,volumeFraction)*10. 

BalancedResourceAllocation 选择的，其实是调度完成后，所有节点里各种资源分配最均衡的那个节点，从而避免一个节点上 CPU 被大量分配、而 Memory 大量剩余的情况。

### NodeAffinityPriority， TaintTolerationPriority， InterPodAffinityPriority

还有 NodeAffinityPriority、TaintTolerationPriority 和 InterPodAffinityPriority 这三种 Priority。顾名思义，它们与前面的 PodMatchNodeSelector、PodToleratesNodeTaints和 PodAffinityPredicate 这三个 Predicate 的含义和计算方法是类似的。但是作为 Priority，一个 Node 满足上述规则的字段数目越多，它的得分就会越高。

### ImageLocalityPriority

如果待调度 Pod 需要使用的镜像很大，并且已经存在于某些 Node上，那么这些 Node 的得分就会比较高。

ImageLocalityPriorityMap is a priority function that favors nodes that already have requested pod container's images.
It will detect whether the requested images are present on a node, and then calculate a score ranging from 0 to 10
based on the total size of those images.
- If none of the images are present, this node will be given the lowest priority.
- If some of the images are present on a node, the larger their sizes' sum, the higher the node's priority.

在实际的执行过程中，调度器里关于集群和 Pod 的信息都已经缓存化，所以算法执行比较快