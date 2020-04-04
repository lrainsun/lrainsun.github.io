---
layout: post
title:  "Kubernetes调度代码解析"
date:   2019-12-20 22:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Scheduler
excerpt: Kubernetes调度代码解析
mathjax: true
typora-root-url: ../
---

# 入口

kubernetes/cmd/kube-scheduler/scheduler.go

```go
func main() {
   rand.Seed(time.Now().UnixNano())

   command := app.NewSchedulerCommand()

   // TODO: once we switch everything over to Cobra commands, we can go back to calling
   // utilflag.InitFlags() (by removing its pflag.Parse() call). For now, we have to set the
   // normalize func and add the go flag set by hand.
   pflag.CommandLine.SetNormalizeFunc(cliflag.WordSepNormalizeFunc)
   // utilflag.InitFlags()
   logs.InitLogs()
   defer logs.FlushLogs()

   if err := command.Execute(); err != nil {
      os.Exit(1)
   }
}
```

运行命令run scheduler

```go
// runCommand runs the scheduler.
func runCommand(cmd *cobra.Command, args []string, opts *options.Options, registryOptions ...Option) error {
   // Get the completed config
   cc := c.Complete()

   // Apply algorithms based on feature gates.
   // TODO: make configurable?  根据当前的feature gates对调度的算法做一些裁剪
   algorithmprovider.ApplyFeatureGates()

   // Configz registration.  注册配置
   if cz, err := configz.New("componentconfig"); err == nil {
      cz.Set(cc.ComponentConfig)
   } else {
      return fmt.Errorf("unable to register configz: %s", err)
   }

   ctx, cancel := context.WithCancel(context.Background())
   defer cancel()
   //启动scheduler
   return Run(ctx, cc, registryOptions...)
}
```

Run函数

```go
// Run executes the scheduler based on the given configuration. It only returns on error or when context is done.
func Run(ctx context.Context, cc schedulerserverconfig.CompletedConfig, 
   // Create the scheduler. 构造一个scheduler
   // 构造调度的预选策略列表、优选策略列表、为各个informer关联handler处理函数
   sched, err := scheduler.New(cc.Client,
      cc.InformerFactory,
      cc.PodInformer,
      cc.Recorder,
      cc.ComponentConfig.AlgorithmSource,
      ctx.Done(),
      scheduler.WithName(cc.ComponentConfig.SchedulerName),
      scheduler.WithHardPodAffinitySymmetricWeight(cc.ComponentConfig.HardPodAffinitySymmetricWeight),
      scheduler.WithPreemptionDisabled(cc.ComponentConfig.DisablePreemption),
      scheduler.WithPercentageOfNodesToScore(cc.ComponentConfig.PercentageOfNodesToScore),
      scheduler.WithBindTimeoutSeconds(*cc.ComponentConfig.BindTimeoutSeconds),
      scheduler.WithFrameworkOutOfTreeRegistry(outOfTreeRegistry),
      scheduler.WithFrameworkPlugins(cc.ComponentConfig.Plugins),
      scheduler.WithFrameworkPluginConfig(cc.ComponentConfig.PluginConfig),
      scheduler.WithPodMaxBackoffSeconds(*cc.ComponentConfig.PodMaxBackoffSeconds),
      scheduler.WithPodInitialBackoffSeconds(*cc.ComponentConfig.PodInitialBackoffSeconds),
   )
   if err != nil {
      return err
   }

   // Start all informers. 启动pod informer，监听pod的变化
   go cc.PodInformer.Informer().Run(ctx.Done())
   cc.InformerFactory.Start(ctx.Done())

   // Wait for all caches to sync before scheduling.
   cc.InformerFactory.WaitForCacheSync(ctx.Done())

   // If leader election is enabled, runCommand via LeaderElector until done and exit.
   if cc.LeaderElection != nil {
      cc.LeaderElection.Callbacks = leaderelection.LeaderCallbacks{
         OnStartedLeading: sched.Run,
         OnStoppedLeading: func() {
            klog.Fatalf("leaderelection lost")
         },
      }
      leaderElector, err := leaderelection.NewLeaderElector(*cc.LeaderElection)
      if err != nil {
         return fmt.Errorf("couldn't create leader elector: %v", err)
      }

      leaderElector.Run(ctx)

      return fmt.Errorf("lost lease")
   }

   // Leader election is disabled, so runCommand inline until done.
   sched.Run(ctx)
   return fmt.Errorf("finished without leader elect")
}
```

```go
// Run begins watching and scheduling. It waits for cache to be synced, then starts scheduling and blocked until the context is done.
func (sched *Scheduler) Run(ctx context.Context) {
   if !cache.WaitForCacheSync(ctx.Done(), sched.scheduledPodsHasSynced) {
      return
   }

   wait.UntilWithContext(ctx, sched.scheduleOne, 0)
}
```

最终调scheduleOne

```go
// scheduleOne does the entire scheduling workflow for a single pod.  It is serialized on the scheduling algorithm's host fitting.
func (sched *Scheduler) scheduleOne(ctx context.Context) {
   fwk := sched.Framework
   //从待调度队列拿到一个待调度的pod
   podInfo := sched.NextPod()
   pod := podInfo.Pod
   // pod could be nil when schedulerQueue is closed
   if pod == nil {
      return
   }
   //跳过删除的pod
   if pod.DeletionTimestamp != nil {
      sched.Recorder.Eventf(pod, nil, v1.EventTypeWarning, "FailedScheduling", "Scheduling", "skip schedule deleting pod: %v/%v", pod.Namespace, pod.Name)
      klog.V(3).Infof("Skip schedule deleting pod: %v/%v", pod.Namespace, pod.Name)
      return
   }

   // Synchronously attempt to find a fit for the pod. 执行调度算法
   start := time.Now()
   state := framework.NewCycleState()
   scheduleResult, err := sched.Algorithm.Schedule(ctx, state, pod)

   // Tell the cache to assume that a pod now is running on a given node, even though it hasn't been bound yet. 
   // This allows us to keep scheduling without waiting on binding to occur.
   assumedPodInfo := podInfo.DeepCopy()
   assumedPod := assumedPodInfo.Pod

   // Assume volumes first before assuming the pod.
   //
   // If all volumes are completely bound, then allBound is true and binding will be skipped.
   //
   // Otherwise, binding of volumes is started after the pod is assumed, but before pod binding.
   // 为Pod中那些还没被Bound的PVCs寻找合适的PVs，并更新PV cache，完成PVs和PVCs的prebound操作
   // This function modifies 'assumedPod' if volume binding is required.
   allBound, err := sched.VolumeBinder.Binder.AssumePodVolumes(assumedPod, scheduleResult.SuggestedHost)

   // Run "reserve" plugins. 运行配置的reserve plugins，如果失败就不再继续schedule
   if sts := fwk.RunReservePlugins(ctx, state, assumedPod, scheduleResult.SuggestedHost); 
  
   // 修改NodeName为schedule的结果node
   // assume modifies `assumedPod` by setting NodeName=scheduleResult.SuggestedHost
   err = sched.assume(assumedPod, scheduleResult.SuggestedHost)

   // bind the pod to its host asynchronously (we can do this b/c of the assumption step above). 异步绑定pod到指定host
   go func() {
      // Bind volumes first before Pod  先绑定volumes
      if !allBound {
         err := sched.bindVolumes(assumedPod)
      }

      // Run "permit" plugins.  
      permitStatus := fwk.RunPermitPlugins(ctx, state, assumedPod, scheduleResult.SuggestedHost)

      // Run "prebind" plugins.
      preBindStatus := fwk.RunPreBindPlugins(ctx, state, assumedPod, scheduleResult.SuggestedHost)

      // 绑定
      err := sched.bind(ctx, assumedPod, scheduleResult.SuggestedHost, state)

     // Run "postbind" plugins.
         fwk.RunPostBindPlugins(ctx, state, assumedPod, scheduleResult.SuggestedHost)
   }
}
```

sched.Algorithm.Schedule

```go
// Schedule tries to schedule the given pod to one of the nodes in the node list.
// If it succeeds, it will return the name of the node.
// If it fails, it will return a FitError error with reasons.
func (g *genericScheduler) Schedule(ctx context.Context, state *framework.CycleState, pod *v1.Pod) (result ScheduleResult, err error) {
   // 做一些基本的检查，pod里定义的pvc是不是存在
   if err := podPassesBasicChecks(pod, g.pvcLister); err != nil {
      return result, err
   }
   trace.Step("Basic checks done")
   // 更新当前node info到cache里
   if err := g.snapshot(); err != nil {
      return result, err
   }

   // Run "prefilter" plugins. 执行preFilterPlugins
   preFilterStatus := g.framework.RunPreFilterPlugins(ctx, state, pod)
 
   // filter符合predicate的node，会启动16个goroutine并调用podFitsOnNode来filter，并发地为集群里的所有 Node 计算 Predicates
   // workqueue.ParallelizeUntil(ctx, 16, allNodes, checkNode)
   filteredNodes, failedPredicateMap, filteredNodesStatuses, err := g.findNodesThatFit(ctx, state, pod)

   // Run "postfilter" plugins.
   postfilterStatus := g.framework.RunPostFilterPlugins(ctx, state, pod, filteredNodes, filteredNodesStatuses)

   if len(filteredNodes) == 0 {
      return result, &FitError{
         Pod:                   pod,
         NumAllNodes:           len(g.nodeInfoSnapshot.NodeInfoList),
         FailedPredicates:      failedPredicateMap,
         FilteredNodesStatuses: filteredNodesStatuses,
      }
   }

   startPriorityEvalTime := time.Now()
   // When only one node after predicate, just use it. 
   // 如果只返回了一个node，跳过priority直接返回结果
   if len(filteredNodes) == 1 {
      return ScheduleResult{
         SuggestedHost:  filteredNodes[0].Name,
         EvaluatedNodes: 1 + len(failedPredicateMap) + len(filteredNodesStatuses),
         FeasibleNodes:  1,
      }, nil
   }

   // 并发为每个node打分，0-10分，0分最低，每个priority func有自己的权重，一个priority的得分是score*weight，再把所有的加起来得到总分
   //workqueue.ParallelizeUntil(context.TODO(), 16, len(nodes), func(index int)
   priorityList, err := g.prioritizeNodes(ctx, state, pod, metaPrioritiesInterface, filteredNodes)

   // 从返回的list中挑选最高分得主
   host, err := g.selectHost(priorityList)
   trace.Step("Prioritizing done")

   // 返回schedule结果
   return ScheduleResult{
      SuggestedHost:  host,
      EvaluatedNodes: len(filteredNodes) + len(failedPredicateMap) + len(filteredNodesStatuses),
      FeasibleNodes:  len(filteredNodes),
   }, err
}
```

sched.VolumeBinder.Binder.AssumePodVolumes

```go
// AssumePodVolumes will take the cached matching PVs and PVCs to provision
// in podBindingCache for the chosen node, and:
// 1. Update the pvCache with the new prebound PV.
// 2. Update the pvcCache with the new PVCs with annotations set
// 3. Update podBindingCache again with cached API updates for PVs and PVCs.
func (b *volumeBinder) AssumePodVolumes(assumedPod *v1.Pod, nodeName string) (allFullyBound bool, err error) {
   podName := getPodName(assumedPod)

   // 所有的pvc都绑定了，直接返回
   if allBound := b.arePodVolumesBound(assumedPod); allBound {
      klog.V(4).Infof("AssumePodVolumes for pod %q, node %q: all PVCs bound and nothing to do", podName, nodeName)
      return true, nil
   }

   assumedPod.Spec.NodeName = nodeName

   // 返回pod, node的cached bindings
   claimsToBind := b.podBindingCache.GetBindings(assumedPod, nodeName)
   claimsToProvision := b.podBindingCache.GetProvisionedPVCs(assumedPod, nodeName)

   // Assume PV
   newBindings := []*bindingInfo{}
   for _, binding := range claimsToBind {
      newPV, dirty, err := pvutil.GetBindVolumeToClaim(binding.pv, binding.pvc)
      klog.V(5).Infof("AssumePodVolumes: GetBindVolumeToClaim for pod %q, PV %q, PVC %q.  newPV %p, dirty %v, err: %v",
         podName,
         binding.pv.Name,
         binding.pvc.Name,
         newPV,
         dirty,
         err)
      if err != nil {
         b.revertAssumedPVs(newBindings)
         return false, err
      }
      // TODO: can we assume everytime?
      if dirty {
         err = b.pvCache.Assume(newPV)
         if err != nil {
            b.revertAssumedPVs(newBindings)
            return false, err
         }
      }
      newBindings = append(newBindings, &bindingInfo{pv: newPV, pvc: binding.pvc})
   }

   // Assume PVCs
   newProvisionedPVCs := []*v1.PersistentVolumeClaim{}
   for _, claim := range claimsToProvision {
      // The claims from method args can be pointing to watcher cache. We must not
      // modify these, therefore create a copy.
      claimClone := claim.DeepCopy()
      metav1.SetMetaDataAnnotation(&claimClone.ObjectMeta, pvutil.AnnSelectedNode, nodeName)
      err = b.pvcCache.Assume(claimClone)
      if err != nil {
         b.revertAssumedPVs(newBindings)
         b.revertAssumedPVCs(newProvisionedPVCs)
         return
      }

      newProvisionedPVCs = append(newProvisionedPVCs, claimClone)
   }

   // Update cache with the assumed pvcs and pvs
   // Even if length is zero, update the cache with an empty slice to indicate that no
   // operations are needed
   b.podBindingCache.UpdateBindings(assumedPod, nodeName, newBindings, newProvisionedPVCs)

   return
}
```

sched.assume

```go
// assume signals to the cache that a pod is already in the cache, so that binding can be asynchronous.
// assume modifies `assumed`.
func (sched *Scheduler) assume(assumed *v1.Pod, host string) error {
   // Optimistically assume that the binding will succeed and send it to apiserver
   // in the background.
   // If the binding fails, scheduler will release resources allocated to assumed pod
   // immediately.
   assumed.Spec.NodeName = host

   if err := sched.SchedulerCache.AssumePod(assumed); err != nil {
      klog.Errorf("scheduler cache AssumePod failed: %v", err)
      return err
   }
   // if "assumed" is a nominated pod, we should remove it from internal cache
   if sched.SchedulingQueue != nil {
      sched.SchedulingQueue.DeleteNominatedPodIfExists(assumed)
   }

   return nil
}
```

异步绑定，在pod之前先绑定volume sched.bindVolumes