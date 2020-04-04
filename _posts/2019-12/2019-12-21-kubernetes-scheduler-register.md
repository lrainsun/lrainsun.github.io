---
layout: post
title:  "Kubernetes调度算法注册"
date:   2019-12-21 22:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Scheduler
excerpt: Kubernetes调度算法注册
mathjax: true
typora-root-url: ../
---

# 调度算法注册

调度算法的注册在kubernetes/pkg/scheduler/algorithmprovider下

![image-20191221112338177](/assets/images/image-20191221112338177.png)

而具体的算法定义在kubernetes/pkg/scheduler/algorithm下，predicates跟priorities的算法都在这下面

![image-20191221112401876](/assets/images/image-20191221112401876.png)

# ApplyFeatureGates

上次学习到run scheduler的时候会有 `algorithmprovider.ApplyFeatureGates()`，根据当前的feature gates对调度的算法做一些裁剪

我们看下default algorithmprovider的这个函数，在kubernetes/pkg/scheduler/algorithmprovider/defaults/defaults.go

```go
// ApplyFeatureGates applies algorithm by feature gates.
// The returned function is used to restore the state of registered predicates/priorities
// when this function is called, and should be called in tests which may modify the value
// of a feature gate temporarily.
func ApplyFeatureGates() (restore func()) {
   snapshot := scheduler.RegisteredPredicatesAndPrioritiesSnapshot()

   // Only register EvenPodsSpread predicate & priority if the feature is enabled
   // 如果支持EvenPodsSpread feature（描述符合条件的一组 Pod 在指定TopologyKey上的打散要求），那么注册对应的调度算法
   if utilfeature.DefaultFeatureGate.Enabled(features.EvenPodsSpread) {
      klog.Infof("Registering EvenPodsSpread predicate and priority function")
      // register predicate
      scheduler.InsertPredicateKeyToAlgorithmProviderMap(predicates.EvenPodsSpreadPred)
      scheduler.RegisterFitPredicate(predicates.EvenPodsSpreadPred, predicates.EvenPodsSpreadPredicate)
      // register priority
      scheduler.InsertPriorityKeyToAlgorithmProviderMap(priorities.EvenPodsSpreadPriority)
      scheduler.RegisterPriorityMapReduceFunction(
         priorities.EvenPodsSpreadPriority,
         priorities.CalculateEvenPodsSpreadPriorityMap,
         priorities.CalculateEvenPodsSpreadPriorityReduce,
         1,
      )
   }

   // Prioritizes nodes that satisfy pod's resource limits
   // ResourceLimitsPriorityFunction -- 启用一个调度器优先权函数，该函数为至少满足输入 Pod 的 cpu 和内存 limit 其中一项的节点分配一个最低可能为 1 的分数。 目的是为了打破相同分数节点的关系
   if utilfeature.DefaultFeatureGate.Enabled(features.ResourceLimitsPriorityFunction) {
      klog.Infof("Registering resourcelimits priority function")
      scheduler.RegisterPriorityMapReduceFunction(priorities.ResourceLimitsPriority, priorities.ResourceLimitsPriorityMap, nil, 1)
      // Register the priority function to specific provider too.
      scheduler.InsertPriorityKeyToAlgorithmProviderMap(scheduler.RegisterPriorityMapReduceFunction(priorities.ResourceLimitsPriority, priorities.ResourceLimitsPriorityMap, nil, 1))
   }

   restore = func() {
      scheduler.ApplyPredicatesAndPriorities(snapshot)
   }
   return
}
```

# Init

在调用到algorithmprovider的时候，会执行default的algorithmprovider的初始化函数，注册predicates和priorities算法

kubernetes/pkg/scheduler/algorithmprovider/defaults/defaults.go

```go
func init() {
   registerAlgorithmProvider(defaultPredicates(), defaultPriorities())
}

func defaultPredicates() sets.String {
	return sets.NewString(
		predicates.NoVolumeZoneConflictPred,
		predicates.MaxCSIVolumeCountPred,
		predicates.MatchInterPodAffinityPred,
		predicates.NoDiskConflictPred,
		predicates.GeneralPred,
		predicates.PodToleratesNodeTaintsPred,
		predicates.CheckVolumeBindingPred,
		predicates.CheckNodeUnschedulablePred,
	)
}

func defaultPriorities() sets.String {
	return sets.NewString(
		priorities.SelectorSpreadPriority,
		priorities.InterPodAffinityPriority,
		priorities.LeastRequestedPriority,
		priorities.BalancedResourceAllocation,
		priorities.NodePreferAvoidPodsPriority,
		priorities.NodeAffinityPriority,
		priorities.TaintTolerationPriority,
		priorities.ImageLocalityPriority,
	)
}
```

registerAlgorithmProvider，注册一个DefaultProvider，ClusterAutoscalerProvider

```go
func registerAlgorithmProvider(predSet, priSet sets.String) {
   // Registers algorithm providers. By default we use 'DefaultProvider', but user can specify one to be used
   // by specifying flag.
   scheduler.RegisterAlgorithmProvider(scheduler.DefaultProvider, predSet, priSet)
   // Cluster autoscaler friendly scheduling algorithm.
   scheduler.RegisterAlgorithmProvider(ClusterAutoscalerProvider, predSet,
      copyAndReplace(priSet, priorities.LeastRequestedPriority, priorities.MostRequestedPriority))
}
```

# predicates

register_predicates.go

在init注册部分注册的predicates

| pridicates            | func                 | details                                                      |
| --------------------- | -------------------- | ------------------------------------------------------------ |
| PodFitsPorts          | PodFitsHostPorts     | 被PodFitsHostPorts替代了，放在这里为了跟1.0兼容              |
| PodFitsHostPortsPred  | PodFitsHostPorts     | 判断是否与宿主机的端口冲突（default pridicate，被GeneralPredicates调用） |
| PodFitsResourcesPred  | PodFitsResources     | 判断node资源是否够（default pridicate，被GeneralPredicates调用） |
| HostNamePred          | PodFitsHost          | 判断pod所指定调度的节点是否是当前的节点（default pridicate，被GeneralPredicates调用） |
| MatchNodeSelectorPred | PodMatchNodeSelector | 判断pod指定的node selector是否匹配当前的node                 |

默认的predicates

| pridicates                       | func                            | details                                              |
| -------------------------------- | ------------------------------- | ---------------------------------------------------- |
| NoVolumeZoneConflictPred         | NewVolumeZonePredicate          | 判断pod使用到的volume是否有节点的要求。目前只支持pvc |
| predicates.MaxCSIVolumeCountPred | NewCSIMaxVolumeLimitPredicate   | 判断CSIVolume是否达到上限了                          |
| MatchInterPodAffinityPred        | NewPodAffinityPredicate         | 匹配pod的亲缘性                                      |
| NoDiskConflictPred               | NoDiskConflict                  | 判断是否有disk volumes的冲突                         |
| GeneralPred                      | GeneralPredicates               | 通用的预选策略                                       |
| PodToleratesNodeTaintsPred       | PodToleratesNodeTaints          | 判断pod是否可以容忍节点的taints                      |
| CheckVolumeBindingPred           | NewVolumeBindingPredicate       | 判断是否有volume拓扑的要求                           |
| CheckNodeUnschedulablePred       | CheckNodeUnschedulablePredicate | pod是否可以容忍unschedulable的node                   |

```go
func init() {
   // Register functions that extract metadata used by predicates computations.
   scheduler.RegisterPredicateMetadataProducer(predicates.GetPredicateMetadata)

   // IMPORTANT NOTES for predicate developers:
   // Registers predicates and priorities that are not enabled by default, but user can pick when creating their
   // own set of priorities/predicates.

   // PodFitsPorts has been replaced by PodFitsHostPorts for better user understanding.
   // For backwards compatibility with 1.0, PodFitsPorts is registered as well.
   scheduler.RegisterFitPredicate("PodFitsPorts", predicates.PodFitsHostPorts)
   // Fit is defined based on the absence of port conflicts.
   // This predicate is actually a default predicate, because it is invoked from
   // predicates.GeneralPredicates()
   scheduler.RegisterFitPredicate(predicates.PodFitsHostPortsPred, predicates.PodFitsHostPorts)
   // Fit is determined by resource availability.
   // This predicate is actually a default predicate, because it is invoked from
   // predicates.GeneralPredicates()
   scheduler.RegisterFitPredicate(predicates.PodFitsResourcesPred, predicates.PodFitsResources)
   // Fit is determined by the presence of the Host parameter and a string match
   // This predicate is actually a default predicate, because it is invoked from
   // predicates.GeneralPredicates()
   scheduler.RegisterFitPredicate(predicates.HostNamePred, predicates.PodFitsHost)
   // Fit is determined by node selector query.
   scheduler.RegisterFitPredicate(predicates.MatchNodeSelectorPred, predicates.PodMatchNodeSelector)

   // Fit is determined by volume zone requirements.
   scheduler.RegisterFitPredicateFactory(
      predicates.NoVolumeZoneConflictPred,
      func(args scheduler.PluginFactoryArgs) predicates.FitPredicate {
         return predicates.NewVolumeZonePredicate(args.PVLister, args.PVCLister, args.StorageClassLister)
      },
   )
   // Fit is determined by whether or not there would be too many AWS EBS volumes attached to the node
   scheduler.RegisterFitPredicateFactory(
      predicates.MaxEBSVolumeCountPred,
      func(args scheduler.PluginFactoryArgs) predicates.FitPredicate {
         return predicates.NewMaxPDVolumeCountPredicate(predicates.EBSVolumeFilterType, args.CSINodeLister, args.StorageClassLister, args.PVLister, args.PVCLister)
      },
   )
   // Fit is determined by whether or not there would be too many GCE PD volumes attached to the node
   scheduler.RegisterFitPredicateFactory(
      predicates.MaxGCEPDVolumeCountPred,
      func(args scheduler.PluginFactoryArgs) predicates.FitPredicate {
         return predicates.NewMaxPDVolumeCountPredicate(predicates.GCEPDVolumeFilterType, args.CSINodeLister, args.StorageClassLister, args.PVLister, args.PVCLister)
      },
   )
   // Fit is determined by whether or not there would be too many Azure Disk volumes attached to the node
   scheduler.RegisterFitPredicateFactory(
      predicates.MaxAzureDiskVolumeCountPred,
      func(args scheduler.PluginFactoryArgs) predicates.FitPredicate {
         return predicates.NewMaxPDVolumeCountPredicate(predicates.AzureDiskVolumeFilterType, args.CSINodeLister, args.StorageClassLister, args.PVLister, args.PVCLister)
      },
   )
   scheduler.RegisterFitPredicateFactory(
      predicates.MaxCSIVolumeCountPred,
      func(args scheduler.PluginFactoryArgs) predicates.FitPredicate {
         return predicates.NewCSIMaxVolumeLimitPredicate(args.CSINodeLister, args.PVLister, args.PVCLister, args.StorageClassLister)
      },
   )
   scheduler.RegisterFitPredicateFactory(
      predicates.MaxCinderVolumeCountPred,
      func(args scheduler.PluginFactoryArgs) predicates.FitPredicate {
         return predicates.NewMaxPDVolumeCountPredicate(predicates.CinderVolumeFilterType, args.CSINodeLister, args.StorageClassLister, args.PVLister, args.PVCLister)
      },
   )

   // Fit is determined by inter-pod affinity.
   scheduler.RegisterFitPredicateFactory(
      predicates.MatchInterPodAffinityPred,
      func(args scheduler.PluginFactoryArgs) predicates.FitPredicate {
         return predicates.NewPodAffinityPredicate(args.NodeInfoLister, args.PodLister)
      },
   )

   // Fit is determined by non-conflicting disk volumes.
   scheduler.RegisterFitPredicate(predicates.NoDiskConflictPred, predicates.NoDiskConflict)

   // GeneralPredicates are the predicates that are enforced by all Kubernetes components
   // (e.g. kubelet and all schedulers)
   scheduler.RegisterFitPredicate(predicates.GeneralPred, predicates.GeneralPredicates)

   // Fit is determined based on whether a pod can tolerate all of the node's taints
   scheduler.RegisterMandatoryFitPredicate(predicates.PodToleratesNodeTaintsPred, predicates.PodToleratesNodeTaints)

   // Fit is determined based on whether a pod can tolerate unschedulable of node
   scheduler.RegisterMandatoryFitPredicate(predicates.CheckNodeUnschedulablePred, predicates.CheckNodeUnschedulablePredicate)

   // Fit is determined by volume topology requirements.
   scheduler.RegisterFitPredicateFactory(
      predicates.CheckVolumeBindingPred,
      func(args scheduler.PluginFactoryArgs) predicates.FitPredicate {
         return predicates.NewVolumeBindingPredicate(args.VolumeBinder)
      },
   )
}
```

# priorities

register_priorities.go

在init注册部分注册的priorities

```go
// EqualPriority is a prioritizer function that gives an equal weight of one to all nodes
// Register the priority function so that its available
// but do not include it as part of the default priorities
scheduler.RegisterPriorityMapReduceFunction(priorities.EqualPriority, core.EqualPriorityMap, nil, 1)
// Optional, cluster-autoscaler friendly priority function - give used nodes higher priority.
scheduler.RegisterPriorityMapReduceFunction(priorities.MostRequestedPriority, priorities.MostRequestedPriorityMap, nil, 1)
scheduler.RegisterPriorityMapReduceFunction(
   priorities.RequestedToCapacityRatioPriority,
   priorities.RequestedToCapacityRatioResourceAllocationPriorityDefault().PriorityMap,
   nil,
   1)
```

default的priorities

| priorities                  | func                                    | details                                            |
| --------------------------- | --------------------------------------- | -------------------------------------------------- |
| SelectorSpreadPriority      | NewSelectorSpreadPriority               | 属于相同service和rs下的pod尽量分布在不同的node上   |
| InterPodAffinityPriority    | NewInterPodAffinityPriority             | 根据pod的亲缘性，将相同拓扑域中的pod放在同一个节点 |
| LeastRequestedPriority      | LeastRequestedPriorityMap               | 按最少请求的利用率对节点进行优先级排序             |
| BalancedResourceAllocation  | BalancedResourceAllocationMap           | 实现资源的平衡使用                                 |
| NodePreferAvoidPodsPriority | CalculateNodePreferAvoidPodsPriorityMap | 将此权重设置为足以覆盖所有其他优先级函数           |
| NodeAffinityPriority        | CalculateNodeAffinityPriorityMap        | pod指定label节点调度，来匹配node亲缘性             |
| TaintTolerationPriority     | ComputeTaintTolerationPriorityMap       | pod有设置tolerate属性来容忍node的taint             |
| ImageLocalityPriority       | ImageLocalityPriorityMap                | 根据节点上是否有该pod使用到的镜像打分              |

```go
func init() {
   // Register functions that extract metadata used by priorities computations.
   scheduler.RegisterPriorityMetadataProducerFactory(
      func(args scheduler.PluginFactoryArgs) priorities.PriorityMetadataProducer {
         return priorities.NewPriorityMetadataFactory(args.ServiceLister, args.ControllerLister, args.ReplicaSetLister, args.StatefulSetLister)
      })

   // ServiceSpreadingPriority is a priority config factory that spreads pods by minimizing
   // the number of pods (belonging to the same service) on the same node.
   // Register the factory so that it's available, but do not include it as part of the default priorities
   // Largely replaced by "SelectorSpreadPriority", but registered for backward compatibility with 1.0
   scheduler.RegisterPriorityConfigFactory(
      priorities.ServiceSpreadingPriority,
      scheduler.PriorityConfigFactory{
         MapReduceFunction: func(args scheduler.PluginFactoryArgs) (priorities.PriorityMapFunction, priorities.PriorityReduceFunction) {
            return priorities.NewSelectorSpreadPriority(args.ServiceLister, algorithm.EmptyControllerLister{}, algorithm.EmptyReplicaSetLister{}, algorithm.EmptyStatefulSetLister{})
         },
         Weight: 1,
      },
   )
   // EqualPriority is a prioritizer function that gives an equal weight of one to all nodes
   // Register the priority function so that its available
   // but do not include it as part of the default priorities
   scheduler.RegisterPriorityMapReduceFunction(priorities.EqualPriority, core.EqualPriorityMap, nil, 1)
   // Optional, cluster-autoscaler friendly priority function - give used nodes higher priority.
   scheduler.RegisterPriorityMapReduceFunction(priorities.MostRequestedPriority, priorities.MostRequestedPriorityMap, nil, 1)
   scheduler.RegisterPriorityMapReduceFunction(
      priorities.RequestedToCapacityRatioPriority,
      priorities.RequestedToCapacityRatioResourceAllocationPriorityDefault().PriorityMap,
      nil,
      1)
   // spreads pods by minimizing the number of pods (belonging to the same service or replication controller) on the same node.
   scheduler.RegisterPriorityConfigFactory(
      priorities.SelectorSpreadPriority,
      scheduler.PriorityConfigFactory{
         MapReduceFunction: func(args scheduler.PluginFactoryArgs) (priorities.PriorityMapFunction, priorities.PriorityReduceFunction) {
            return priorities.NewSelectorSpreadPriority(args.ServiceLister, args.ControllerLister, args.ReplicaSetLister, args.StatefulSetLister)
         },
         Weight: 1,
      },
   )
   // pods should be placed in the same topological domain (e.g. same node, same rack, same zone, same power domain, etc.)
   // as some other pods, or, conversely, should not be placed in the same topological domain as some other pods.
   scheduler.RegisterPriorityConfigFactory(
      priorities.InterPodAffinityPriority,
      scheduler.PriorityConfigFactory{
         Function: func(args scheduler.PluginFactoryArgs) priorities.PriorityFunction {
            return priorities.NewInterPodAffinityPriority(args.HardPodAffinitySymmetricWeight)
         },
         Weight: 1,
      },
   )

   // Prioritize nodes by least requested utilization.
   scheduler.RegisterPriorityMapReduceFunction(priorities.LeastRequestedPriority, priorities.LeastRequestedPriorityMap, nil, 1)

   // Prioritizes nodes to help achieve balanced resource usage
   scheduler.RegisterPriorityMapReduceFunction(priorities.BalancedResourceAllocation, priorities.BalancedResourceAllocationMap, nil, 1)

   // Set this weight large enough to override all other priority functions.
   // TODO: Figure out a better way to do this, maybe at same time as fixing #24720.
   scheduler.RegisterPriorityMapReduceFunction(priorities.NodePreferAvoidPodsPriority, priorities.CalculateNodePreferAvoidPodsPriorityMap, nil, 10000)

   // Prioritizes nodes that have labels matching NodeAffinity
   scheduler.RegisterPriorityMapReduceFunction(priorities.NodeAffinityPriority, priorities.CalculateNodeAffinityPriorityMap, priorities.CalculateNodeAffinityPriorityReduce, 1)

   // Prioritizes nodes that marked with taint which pod can tolerate.
   scheduler.RegisterPriorityMapReduceFunction(priorities.TaintTolerationPriority, priorities.ComputeTaintTolerationPriorityMap, priorities.ComputeTaintTolerationPriorityReduce, 1)

   // ImageLocalityPriority prioritizes nodes that have images requested by the pod present.
   scheduler.RegisterPriorityMapReduceFunction(priorities.ImageLocalityPriority, priorities.ImageLocalityPriorityMap, nil, 1)
}
```