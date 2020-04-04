---
layout: post
title:  "Kubernetes抢占机制"
date:   2019-12-22 22:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Scheduler
excerpt: Kubernetes抢占机制
mathjax: true
typora-root-url: ../
---

# 抢占

正常情况下，当一个 Pod 调度失败后，它就会被暂时“搁置”起来，直到 Pod 被更新，或者集群状态发生变化，调度器才会对这个 Pod 进行重新调度。

但在有时候，我们希望的是这样一个场景。当一个高优先级的 Pod 调度失败后，该 Pod 并不会被“搁置”，而是会“挤走”某个 Node 上的一些低优先级的 Pod 。这样就可以保证这个高优先级 Pod 的调度成功，这就是抢占。

在kubernetes里可以定义PriorityClass来定义优先级

```yaml
apiVersion: scheduling.k8s.io/v1beta1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for high priority service pods only."
```

Kubernetes 规定，优先级是一个 32 bit 的整数，最大值不超过 1000000000（10 亿，1billion），并且值越大代表优先级越高。而超出 10 亿的值，其实是被 Kubernetes 保留下来分配给系统 Pod 使用的。显然，这样做的目的，就是保证系统 Pod 不会被用户抢占掉。

high-priority是我们创建的，而system-cluster-critical和system-node-critical是系统pod用的

```shell
[root@rain-kubernetes-1 rain]# kubectl get pc
NAME                      VALUE        GLOBAL-DEFAULT   AGE
high-priority             1000000      false            20m
system-cluster-critical   2000000000   false            59d
system-node-critical      2000001000   false            59d
```

一旦上述 YAML 文件里的 globalDefault 被设置为 true 的话，那就意味着这个PriorityClass 的值会成为系统的默认值。而如果这个值是 false，就表示我们只希望声明使用该PriorityClass 的 Pod 拥有值为 1000000 的优先级，而对于没有声明 PriorityClass 的 Pod 来说，它们的优先级就是 0。

可以创建一个pod使用刚刚定义的PriorityClass

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  priorityClassName: high-priority
```

这样这个pod的优先级就高于其他pod（默认是0）

在schedule的时候，如果failed了，会尝试preempt

```go
scheduleResult, err := sched.Algorithm.Schedule(ctx, state, pod)
if err != nil {
   sched.recordSchedulingFailure(podInfo.DeepCopy(), err, v1.PodReasonUnschedulable, err.Error())
   // Schedule() may have failed because the pod would not fit on any host, so we try to
   // preempt, with the expectation that the next time the pod is tried for scheduling it
   // will fit due to the preemption. It is also possible that a different pod will schedule
   // into the resources that were preempted, but this is harmless.
   if fitError, ok := err.(*core.FitError); ok {
      if sched.DisablePreemption {
         klog.V(3).Infof("Pod priority feature is not enabled or preemption is disabled by scheduler configuration." +
            " No preemption is performed.")
      } else {
         preemptionStartTime := time.Now()
         sched.preempt(ctx, state, fwk, pod, fitError)
      }
   } 
   return
}
```

当一个高优先级的 Pod 调度失败的时候，调度器的抢占能力就会被触发。这时，调度器就会试图从当前集群里寻找一个节点，使得当这个节点上的一个或者多个低优先级 Pod 被删除后，待调度的高优先级 Pod 就可以被调度到这个节点上。这个过程，就是“抢占”这个概念在Kubernetes里的主要体现。

```go
// preempt tries to create room for a pod that has failed to schedule, by preempting lower priority pods if possible.
// If it succeeds, it adds the name of the node where preemption has happened to the pod spec.
// It returns the node name and an error if any.
func (sched *Scheduler) preempt(ctx context.Context, state *framework.CycleState, fwk framework.Framework, preemptor *v1.Pod, scheduleErr error) (string, error) {
   preemptor, err := sched.podPreemptor.getUpdatedPod(preemptor)
   if err != nil {
      klog.Errorf("Error getting the updated preemptor pod object: %v", err)
      return "", err
   }
  //执行抢占，返回抢占成功的node，被抢占的牺牲者victims(一组pod)，需要被移除已提名的 pods(一组)
   node, victims, nominatedPodsToClear, err := sched.Algorithm.Preempt(ctx, state, preemptor, scheduleErr)
   if err != nil {
      klog.Errorf("Error preempting victims to make room for %v/%v: %v", preemptor.Namespace, preemptor.Name, err)
      return "", err
   }
   var nodeName = ""
   if node != nil {
      nodeName = node.Name
      // Update the scheduling queue with the nominated pod information. Without
      // this, there would be a race condition between the next scheduling cycle
      // and the time the scheduler receives a Pod Update for the nominated pod.
      // 更新 scheduler 缓存，为抢占者绑定 nodeName，即设定 pod.Status.NominatedNodeName
      sched.SchedulingQueue.UpdateNominatedPodForNode(preemptor, nodeName)

      // Make a call to update nominated node name of the pod on the API server.
      //将pod info提交到apiserver
      err = sched.podPreemptor.setNominatedNodeName(preemptor, nodeName)
      if err != nil {
         klog.Errorf("Error in preemption process. Cannot set 'NominatedPod' on pod %v/%v: %v", preemptor.Namespace, preemptor.Name, err)
         sched.SchedulingQueue.DeleteNominatedPodIfExists(preemptor)
         return "", err
      }

      for _, victim := range victims {
         //删除被抢占的pod
         if err := sched.podPreemptor.deletePod(victim); err != nil {
            klog.Errorf("Error preempting pod %v/%v: %v", victim.Namespace, victim.Name, err)
            return "", err
         }
         // If the victim is a WaitingPod, send a reject message to the PermitPlugin
         if waitingPod := fwk.GetWaitingPod(victim.UID); waitingPod != nil {
            waitingPod.Reject("preempted")
         }
         sched.Recorder.Eventf(victim, preemptor, v1.EventTypeNormal, "Preempted", "Preempting", "Preempted by %v/%v on node %v", preemptor.Namespace, preemptor.Name, nodeName)

      }
      metrics.PreemptionVictims.Observe(float64(len(victims)))
   }
   // Clearing nominated pods should happen outside of "if node != nil". Node could
   // be nil when a pod with nominated node name is eligible to preempt again,
   // but preemption logic does not find any node for it. In that case Preempt()
   // function of generic_scheduler.go returns the pod itself for removal of
   // the 'NominatedPod' field.
   for _, p := range nominatedPodsToClear {
      //删除被抢占 pods 的 NominatedNodeName 字段
      rErr := sched.podPreemptor.removeNominatedNodeName(p)
      if rErr != nil {
         klog.Errorf("Cannot remove 'NominatedPod' field of pod: %v", rErr)
         // We do not return as this error is not critical.
      }
   }
   return nodeName, err
}
```

抢占逻辑:

```go
// preempt finds nodes with pods that can be preempted to make room for "pod" to
// schedule. It chooses one of the nodes and preempts the pods on the node and
// returns 1) the node, 2) the list of preempted pods if such a node is found,
// 3) A list of pods whose nominated node name should be cleared, and 4) any
// possible error.
// Preempt does not update its snapshot. It uses the same snapshot used in the
// scheduling cycle. This is to avoid a scenario where preempt finds feasible
// nodes without preempting any pod. When there are many pending pods in the
// scheduling queue a nominated pod will go back to the queue and behind
// other pods with the same priority. The nominated pod prevents other pods from
// using the nominated resources and the nominated pod could take a long time
// before it is retried after many other pending pods.
func (g *genericScheduler) Preempt(ctx context.Context, state *framework.CycleState, pod *v1.Pod, scheduleErr error) (*v1.Node, []*v1.Pod, []*v1.Pod, error) {
   // Scheduler may return various types of errors. Consider preemption only if
   // the error is of type FitError. 只考虑schedule失败为FitError的情况
   fitError, ok := scheduleErr.(*FitError)
   if !ok || fitError == nil {
      return nil, nil, nil, nil
   }
   //确定pod是否有抢占其他pod的资格，如果pod 已经抢占了低优先级的 pod，被抢占的 pod 处于 terminating 状态中，则不会继续进行抢占
   if !podEligibleToPreemptOthers(pod, g.nodeInfoSnapshot.NodeInfoMap, g.enableNonPreempting) {
      klog.V(5).Infof("Pod %v/%v is not eligible for more preemption.", pod.Namespace, pod.Name)
      return nil, nil, nil, nil
   }
   if len(g.nodeInfoSnapshot.NodeInfoMap) == 0 {
      return nil, nil, nil, ErrNoNodesAvailable
   }
  //potentialNodes找到的一些nodes: with failed predicates that may be satisfied by removing pods from the node，过滤掉因为nodeselectornotmatch等原因导致predicate失败的那些node，这些原因，即使删除pod也没用
//  var unresolvablePredicateFailureErrors = map[PredicateFailureReason]struct{}{
//	ErrNodeSelectorNotMatch:      {},
//	ErrPodAffinityRulesNotMatch:  {},
//	ErrPodNotMatchHostName:       {},
//	ErrTaintsTolerationsNotMatch: {},
//	ErrNodeLabelPresenceViolated: {},
	// Node conditions won't change when scheduler simulates removal of preemption victims.
	// So, it is pointless to try nodes that have not been able to host the pod due to node
	// conditions. These include ErrNodeNotReady, ErrNodeUnderPIDPressure, ErrNodeUnderMemoryPressure, ....
//	ErrNodeNotReady:            {},
//	ErrNodeNetworkUnavailable:  {},
//	ErrNodeUnderDiskPressure:   {},
//	ErrNodeUnderPIDPressure:    {},
//	ErrNodeUnderMemoryPressure: {},
//	ErrNodeUnschedulable:       {},
//	ErrNodeUnknownCondition:    {},
//	ErrVolumeZoneConflict:      {},
//	ErrVolumeNodeConflict:      {},
//	ErrVolumeBindConflict:      {},
//}
   potentialNodes := nodesWherePreemptionMightHelp(g.nodeInfoSnapshot.NodeInfoMap, fitError)
   if len(potentialNodes) == 0 {
      klog.V(3).Infof("Preemption will not help schedule pod %v/%v on any node.", pod.Namespace, pod.Name)
      // In this case, we should clean-up any existing nominated node name of the pod.
      return nil, nil, []*v1.Pod{pod}, nil
   }
   var (
      pdbs []*policy.PodDisruptionBudget
      err  error
   )
   if g.pdbLister != nil {
      //获取 PodDisruptionBudget 对象 PodDisruptionBudget is an object to define the max disruption that can be caused to a collection of pods
      pdbs, err = g.pdbLister.List(labels.Everything())
      if err != nil {
         return nil, nil, nil, err
      }
   }
   //finds all the nodes with possible victims for preemption in parallel.从预选失败的 node 列表中并发计算可以被抢占的 nodes，得到 nodeToVictims
   nodeToVictims, err := g.selectNodesForPreemption(ctx, state, pod, potentialNodes, pdbs)
   if err != nil {
      return nil, nil, nil, err
   }

   // We will only check nodeToVictims with extenders that support preemption.
   // Extenders which do not support preemption may later prevent preemptor from being scheduled on the nominated
   // node. In that case, scheduler will find a different host for the preemptor in subsequent scheduling cycles.
   // 若声明了 extenders 则调用 extenders 再次过滤 nodeToVictims
   nodeToVictims, err = g.processPreemptionWithExtenders(pod, nodeToVictims)
   if err != nil {
      return nil, nil, nil, err
   }
   // 调用pickOneNodeForPreemption() 从 nodeToVictims 中选出一个节点作为最佳候选人
   // PDB violations 值最小的 node
   // 挑选具有高优先级较少的 node
   // 对每个 node 上所有 victims 的优先级进项累加，选取最小的
   // 如果多个 node 优先级总和相等，选择具有最小 victims  数量的 node
   // 如果多个 node 优先级总和相等，选择具有高优先级且 pod 运行时间最短的
   // 如果依据以上策略仍然选出了多个 node 则直接返回第一个 node
   candidateNode := pickOneNodeForPreemption(nodeToVictims)
   if candidateNode == nil {
      return nil, nil, nil, nil
   }

   // Lower priority pods nominated to run on this node, may no longer fit on
   // this node. So, we should remove their nomination. Removing their
   // nomination updates these pods and moves them to the active queue. It
   // lets scheduler find another place for them.
   //移除低优先级 pod 的 Nominated，更新这些 pod，移动到 activeQ 队列中，让调度器为这些 pod 重新 bind node
   nominatedPods := g.getLowerPriorityNominatedPods(pod, candidateNode.Name)
   if nodeInfo, ok := g.nodeInfoSnapshot.NodeInfoMap[candidateNode.Name]; ok {
      return nodeInfo.Node(), nodeToVictims[candidateNode].Pods, nominatedPods, nil
   }

   return nil, nil, nil, fmt.Errorf(
      "preemption failed: the target node %s has been deleted from scheduler cache",
      candidateNode.Name)
}
```

挑选可以被抢占的node的函数selectNodesForPreemption里调用selectVictimsOnNode来获取每个node上需要牺牲的victims pod

```go
// selectVictimsOnNode finds minimum set of pods on the given node that should
// be preempted in order to make enough room for "pod" to be scheduled. The
// minimum set selected is subject to the constraint that a higher-priority pod
// is never preempted when a lower-priority pod could be (higher/lower relative
// to one another, not relative to the preemptor "pod").
// The algorithm first checks if the pod can be scheduled on the node when all the
// lower priority pods are gone. If so, it sorts all the lower priority pods by
// their priority and then puts them into two groups of those whose PodDisruptionBudget
// will be violated if preempted and other non-violating pods. Both groups are
// sorted by priority. It first tries to reprieve as many PDB violating pods as
// possible and then does them same for non-PDB-violating pods while checking
// that the "pod" can still fit on the node.
// NOTE: This function assumes that it is never called if "pod" cannot be scheduled
// due to pod affinity, node affinity, or node anti-affinity reasons. None of
// these predicates can be satisfied by removing more pods from the node.
func (g *genericScheduler) selectVictimsOnNode(
   ctx context.Context,
   state *framework.CycleState,
   pod *v1.Pod,
   meta predicates.PredicateMetadata,
   nodeInfo *schedulernodeinfo.NodeInfo,
   pdbs []*policy.PodDisruptionBudget,
) ([]*v1.Pod, int, bool) {
   var potentialVictims []*v1.Pod

   removePod := func(rp *v1.Pod) error {
      if err := nodeInfo.RemovePod(rp); err != nil {
         return err
      }
      if meta != nil {
         if err := meta.RemovePod(rp, nodeInfo.Node()); err != nil {
            return err
         }
      }
      status := g.framework.RunPreFilterExtensionRemovePod(ctx, state, pod, rp, nodeInfo)
      if !status.IsSuccess() {
         return status.AsError()
      }
      return nil
   }
   addPod := func(ap *v1.Pod) error {
      nodeInfo.AddPod(ap)
      if meta != nil {
         if err := meta.AddPod(ap, nodeInfo.Node()); err != nil {
            return err
         }
      }
      status := g.framework.RunPreFilterExtensionAddPod(ctx, state, pod, ap, nodeInfo)
      if !status.IsSuccess() {
         return status.AsError()
      }
      return nil
   }
   // As the first step, remove all the lower priority pods from the node and
   // check if the given pod can be scheduled.
   //先删除所有低优先级的pod检查是否能够满足抢占pod的调度需求
   podPriority := podutil.GetPodPriority(pod)
   for _, p := range nodeInfo.Pods() {
      if podutil.GetPodPriority(p) < podPriority {
         potentialVictims = append(potentialVictims, p)
         if err := removePod(p); err != nil {
            return nil, 0, false
         }
      }
   }
   // If the new pod does not fit after removing all the lower priority pods,
   // we are almost done and this node is not suitable for preemption. The only
   // condition that we could check is if the "pod" is failing to schedule due to
   // inter-pod affinity to one or more victims, but we have decided not to
   // support this case for performance reasons. Having affinity to lower
   // priority pods is not a recommended configuration anyway.
   //如果删除掉所有低优先级的pod还不符合要求那么直接过滤掉该node
   if fits, _, _, err := g.podFitsOnNode(ctx, state, pod, meta, nodeInfo, false); !fits {
      if err != nil {
         klog.Warningf("Encountered error while selecting victims on node %v: %v", nodeInfo.Node().Name, err)
      }

      return nil, 0, false
   }
   var victims []*v1.Pod
   numViolatingVictim := 0
   sort.Slice(potentialVictims, func(i, j int) bool { return util.MoreImportantPod(potentialVictims[i], potentialVictims[j]) })
   // Try to reprieve as many pods as possible. We first try to reprieve the PDB
   // violating victims and then other non-violating ones. In both cases, we start
   // from the highest priority victims.
   //尝试尽可能多地删除这些pods，先从 PDB violating victims 中“删除”，再从 PDB non-violating victims 中“删除”
   violatingVictims, nonViolatingVictims := filterPodsWithPDBViolation(potentialVictims, pdbs)
   reprievePod := func(p *v1.Pod) (bool, error) {
      if err := addPod(p); err != nil {
         return false, err
      }
      //调用podFitsOnNode再次执行predicates算法
      fits, _, _, _ := g.podFitsOnNode(ctx, state, pod, meta, nodeInfo, false)
      if !fits {
         if err := removePod(p); err != nil {
            return false, err
         }
         victims = append(victims, p)
         klog.V(5).Infof("Pod %v/%v is a potential preemption victim on node %v.", p.Namespace, p.Name, nodeInfo.Node().Name)
      }
      return fits, nil
   }
   //删除 violatingVictims 中的 pod，同时也记录删除了多少个
   for _, p := range violatingVictims {
      if fits, err := reprievePod(p); err != nil {
         klog.Warningf("Failed to reprieve pod %q: %v", p.Name, err)
         return nil, 0, false
      } else if !fits {
         numViolatingVictim++
      }
   }
   // Now we try to reprieve non-violating victims.
   // 删除 nonViolatingVictims 中的 pod
   for _, p := range nonViolatingVictims {
      if _, err := reprievePod(p); err != nil {
         klog.Warningf("Failed to reprieve pod %q: %v", p.Name, err)
         return nil, 0, false
      }
   }
   return victims, numViolatingVictim, true
}
```

# 抢占机制

当上述抢占过程发生时，抢占者并不会立刻被调度到被抢占的 node 上，调度器只会将抢占者的 status.nominatedNodeName 字段设置为被抢占的 node 的名字。然后，抢占者会重新进入下一个调度周期，在新的调度周期里来决定是不是要运行在被抢占的节点上，当然，即使在下一个调度周期，调度器也不会保证抢占者一定会运行在被抢占的节点上。

这样设计的一个重要原因是调度器只会通过标准的 DELETE API 来删除被抢占的 pod，所以，这些 pod 必然是有一定的“优雅退出”时间（默认是 30s）的。而在这段时间里，其他的节点也是有可能变成可调度的，或者直接有新的节点被添加到这个集群中来。所以，鉴于优雅退出期间集群的可调度性可能会发生的变化，把抢占者交给下一个调度周期再处理，是一个非常合理的选择。而在抢占者等待被调度的过程中，如果有其他更高优先级的 pod 也要抢占同一个节点，那么调度器就会清空原抢占者的 status.nominatedNodeName 字段，从而允许更高优先级的抢占者执行抢占，并且，这也使得原抢占者本身也有机会去重新抢占其他节点。以上这些都是设置 `nominatedNodeName` 字段的主要目的。

# 原理

抢占发生的原因，一定是一个高优先级的 Pod 调度失败。称这个 Pod 为“抢占者”，称被抢占的 Pod 为“牺牲者”（victims）。

## 调度队列

Kubernetes 调度器实现抢占算法的一个最重要的设计，就是在调度队列的实现里，使用了两个不同的队列：

* activeQ：凡是在 activeQ 里的 Pod，都是下一个调度周期需要调度的对象。所以，当你在 Kubernetes 集群里新创建一个 Pod 的时候，调度器会将这个 Pod 入队到activeQ 里面。调度器不断从队列里出队（Pop）一个 Pod 进行调度，实际上都是从 activeQ 里出队的。
* unschedulableQ：存放调度失败的pod

当一个 unschedulableQ 里的 Pod 被更新之后，调度器会自动把这个 Pod 移动到 activeQ 里，从而给这些调度失败的 Pod “重新做人”的机会。

## 寻找牺牲者

调度失败之后，抢占者就会被放进 unschedulableQ 里面。然后，这次失败事件就会触发调度器为抢占者寻找牺牲者的流程。

* 调度器会检查这次失败事件的原因，来确认抢占是不是可以帮助抢占者找到一个新节点。这是因为有很多 Predicates 的失败是不能通过抢占来解决的。比如，PodFitsHost 算法（负责的是，检查 Pod 的 nodeSelector 与 Node 的名字是否匹配），这种情况下，除非Node 的名字发生变化，否则你即使删除再多的 Pod，抢占者也不可能调度成功
* 如果确定抢占可以发生，那么调度器就会把自己缓存的所有节点信息复制一份，然后使用这个副本来模拟抢占过程。
  这里的抢占过程很容易理解。调度器会检查缓存副本里的每一个节点，然后从该节点上最低优先级的 Pod 开始，逐一“删除”这些 Pod。而每删除一个低优先级 Pod，调度器都会检查一下抢占者是否能够运行在该 Node 上。一旦可以运行，调度器就记录下这个 Node 的名字和被删除Pod 的列表，这就是一次抢占过程的结果了。
  当遍历完所有的节点之后，调度器会在上述模拟产生的所有抢占结果里做一个选择，找出最佳结果。而这一步的判断原则，就是尽量减少抢占对整个系统的影响。比如，需要抢占的 Pod 越少越好，需要抢占的 Pod 的优先级越低越好，等等。

## 执行抢占

在得到了最佳的抢占结果之后，这个结果里的 Node，就是即将被抢占的 Node；被删除的Pod 列表，就是牺牲者。所以接下来，调度器就可以真正开始抢占的操作了，这个过程，可以分为三步。

* 调度器会检查牺牲者列表，清理这些 Pod 所携带的 nominatedNodeName 字段。
* 调度器会把抢占者的 nominatedNodeName，设置为被抢占的 Node 的名字。（由于这里更新了抢占着pod，就会触发把该pod重新加入activeQ，在下一个调度周期重新调度，`pod.Status.NominatedNodeName`，表示在NominatedNodeName上发生了抢占，preemptor期望调度在该node上）
* 调度器会开启一个 Goroutine，异步地删除牺牲者。

# 抢占者的predicates

在为某一对 Pod 和 Node 执行 Predicates 算法的时候，如果待检查的 Node 是一个即将被抢占的节点，即：调度队列里有 nominatedNodeName 字段值是该 Node 名字的Pod 存在（可以称之为：“潜在的抢占者”）。那么，调度器就会对这个 Node ，将同样的Predicates 算法运行两遍。

第一遍：调度器会假设上述“潜在的抢占者”已经运行在这个节点上，然后执行 Predicates 算法

这是由于InterPodAntiAffinity 规则关心待考察节点上所有 Pod 之间的互斥关系，所以我们在执行调度算法时必须考虑，如果抢占者已经存在于待考察 Node 上时，待调度 Pod 还能不能调度成功。

在这一步只需要考虑那些优先级等于或者大于待调度 Pod 的抢占者。毕竟对于其他较低优先级 Pod 来说，待调度 Pod 总是可以通过抢占运行在待考察 Node上。

第二遍：调度器会正常执行 Predicates 算法，即：不考虑任何“潜在的抢占者”

们需要执行第二遍 Predicates 算法的原因，则是因为“潜在的抢占者”最后不一定会运行在待考察的 Node 上。

而只有这两遍 Predicates 算法都能通过时，这个 Pod 和 Node 才会被认为是可以绑定（bind）的。