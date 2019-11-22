---
layout: post
title:  "Kubernetes资源管理"
date:   2019-11-22 17:00:00 +0800
categories: Kubernetes
tags: Kubernetes-Architecture
excerpt: Kubernetes架构
mathjax: true
typora-root-url: ../
---

# 资源类型

在 Kubernetes 里，Pod 是最小的原子调度单位。所有跟调度和资源管理相关的属性都应该是属于 Pod 对象的字段。其中最重要的部分，是 Pod 的 CPU 和内存配置。

而由于 Pod 可以由多个 Container 组成，所以 CPU 和内存资源的限额，是要配置在每个Container 的定义上的。这样，Pod 整体的资源配置，就由这些 Container 的配置值累加得到。

资源可以分为两大类：

* 可压缩资源（compressible）
  * 如果系统限制或者缩小容器对可压缩资源的使用的话，只会影响服务对外的服务性能，比如 CPU 就是一种非常典型的可压缩资源。
  * 当可压缩资源不足时，Pod 只会“饥饿”，但不会退出
* 不可压缩资源（incompressible）
  * 对于不可压缩资源来说，资源的紧缺是有可能导致服务对外不可用的，比如内存就是一种非常典型的不可压缩资源。
  * 当不可压缩资源不足时，Pod 就会因为 OOM（Out-Of-Memory）被内核杀掉。

# CPU

不管底层的机器是通过何种方式提供的（物理机 or 虚拟机），一个单位的 CPU 资源都会被标准化为一个标准的 "Kubernetes Compute Unit"。也就是Kubernetes 里为 CPU 设置的单位是“CPU 的个数”。

CPU 资源的基本单位是 millicores，因为 CPU 资源其实准确来讲，指的是 CPU 时间。所以它的基本单位为 millicores，1 个核等于 1000 millicores。也代表了 kubernetes 可以将单位 CPU 时间细分为 1000 份，分配给某个容器。

# Memory

而对于内存资源来说，它的单位自然就是 bytes。Kubernetes 支持你使用 Ei、Pi、Ti、Gi、Mi、Ki（或者 E、P、T、G、M、K）的方式来作为 bytes 的值。

> 注意区分MiB（mebibyte）和 MB（megabyte）的区别。
>
> 1Mi=1024*1024；1M=1000*1000

# 资源管理

我们可以通过get node来获得一个node的资源的使用和实际情况：

```shell
[root@rain-kubernetes-1 kubepods]# kubectl get nodes rain-kubernetes-1 -o yaml
status:
  allocatable:
    cpu: "4"
    ephemeral-storage: "77296556724"
    hugepages-1Gi: "0"
    hugepages-2Mi: "0"
    memory: 3779012Ki
    pods: "110"
  capacity:
    cpu: "4"
    ephemeral-storage: 83872132Ki
    hugepages-1Gi: "0"
    hugepages-2Mi: "0"
    memory: 3881412Ki
    pods: "110"
```

* capacity
  * 资源的实际值
  * 获取资源capacity，是由cadvisor提供的，相当于在 kubelet 内部启动了一个小的 cadvisor。在 kubelet 启动后，这个内部的 cadvisor 子模块也会相应启动，并且获取这台机器上面的各种信息。其中就包括了有关这台机器的资源信息，而这个信息也自然作为这台机器的真实资源信息，通过 kubelet 再上报给 apiserver。

* allocatable

  * 资源的可使用量

  * 在任何情况下，allocatable 是一定小于等于 capacity 的

  * 在 kubernetes 集群中的每一个节点上，除了用户容器还会运行很多其他的重要组件，他们并不是以容器的方式来运行的，这些组件的类型主要分为两个大类：

    * kubernetes daemon：kubernetes 相关的 daemon 程序，比如 kubelet，dockerd 等等。
    * system daemon：和 kubernetes 不相关的其他系统级 daemon 程序，比如 sshd 等等。

    针对这两种进程，kubernetes 分别提供了 Kube-Reserved 和 System-Reserved 特性，可以分别为这两组进程设定一个预留的计算资源量。

# Kubernetes QoS模型

在Kubernetes 中，不同的 requests 和 limits 的设置方式，会将这个 Pod 划分到不同的QoS 级别当中。

在调度的时候，kube-scheduler 只会按照 requests 的值进行计算。而在真正设置 Cgroups 限制的时候，kubelet 则会按照 limits 的值来进行设置。

kubernetes 底层是通过 cgroup 特性来实现资源的隔离与限制的。

针对每一种基本资源（CPU、memory），kubernetes 首先会创建一个根 cgroup 组，作为所有容器 cgroup 的根，名字叫做 kubepods。这个 cgroup 就是用来限制这台计算节点上所有 pod 所使用的资源的。默认情况下这个 kubepods cgroup 组所获取的资源就等同于该计算节点的全部资源。

## Guaranteed

当 Pod 里的每一个 Container 都同时设置了 requests 和 limits，并且 requests 和 limits值相等的时候，这个 Pod 就属于 Guaranteed 类别

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo
  namespace: rain-qos
spec:
  containers:
  - name: qos-demo-ctr
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "700m"
      requests:
        memory: "200Mi"
        cpu: "700m"
```

当这个 Pod 创建之后，它的 qosClass 字段就会被 Kubernetes 自动设置为 Guaranteed。需要注意的是，当 Pod 仅设置了 limits 没有设置 requests 的时候，Kubernetes 会自动为它设置与 limits 相同的 requests 值，所以，这也属于 Guaranteed 情况。

```shell
[root@rain-kubernetes-1 burstable]# kubectl get pods -n rain-qos -o wide
NAME       READY   STATUS    RESTARTS   AGE     IP           NODE                NOMINATED NODE   READINESS GATES
qos-demo   1/1     Running   0          7m18s   10.168.2.2   rain-kubernetes-3   <none>           <none>

[root@rain-kubernetes-3 ~]# docker ps | grep qos-demo-ctr
0313ba90d47d        nginx                              "nginx -g 'daemon of…"   7 minutes ago       Up 7 minutes                            k8s_qos-demo-ctr_qos-demo_rain-qos_866d47f4-4d12-41c1-bbfd-51c81517a8fd_0
```

我们刚起的这个pod的container的cgroup位置在：

```shell
[root@rain-kubernetes-3 ~]# ls -al /sys/fs/cgroup/cpu/kubepods/pod866d47f4-4d12-41c1-bbfd-51c81517a8fd/0313ba90d47dd7dd2c6c2961aab9d28cb150ec079433db479ea7f266fbee4354/
total 0
drwxr-xr-x. 2 root root 0 Nov 22 06:19 .
drwxr-xr-x. 4 root root 0 Nov 22 06:18 ..
-rw-r--r--. 1 root root 0 Nov 22 06:19 cgroup.clone_children
--w--w--w-. 1 root root 0 Nov 22 06:19 cgroup.event_control
-rw-r--r--. 1 root root 0 Nov 22 06:19 cgroup.procs
-r--r--r--. 1 root root 0 Nov 22 06:19 cpuacct.stat
-rw-r--r--. 1 root root 0 Nov 22 06:19 cpuacct.usage
-r--r--r--. 1 root root 0 Nov 22 06:19 cpuacct.usage_percpu
-rw-r--r--. 1 root root 0 Nov 22 06:19 cpu.cfs_period_us
-rw-r--r--. 1 root root 0 Nov 22 06:19 cpu.cfs_quota_us
-rw-r--r--. 1 root root 0 Nov 22 06:19 cpu.rt_period_us
-rw-r--r--. 1 root root 0 Nov 22 06:19 cpu.rt_runtime_us
-rw-r--r--. 1 root root 0 Nov 22 06:19 cpu.shares
-r--r--r--. 1 root root 0 Nov 22 06:19 cpu.stat
-rw-r--r--. 1 root root 0 Nov 22 06:19 notify_on_release
-rw-r--r--. 1 root root 0 Nov 22 06:19 tasks

[root@rain-kubernetes-3 ~]# ls -al /sys/fs/cgroup/memory/kubepods/pod866d47f4-4d12-41c1-bbfd-51c81517a8fd/0313ba90d47dd7dd2c6c2961aab9d28cb150ec079433db479ea7f266fbee4354/
total 0
drwxr-xr-x. 2 root root 0 Nov 22 06:19 .
drwxr-xr-x. 4 root root 0 Nov 22 06:18 ..
-rw-r--r--. 1 root root 0 Nov 22 06:19 cgroup.clone_children
--w--w--w-. 1 root root 0 Nov 22 06:19 cgroup.event_control
-rw-r--r--. 1 root root 0 Nov 22 06:19 cgroup.procs
-rw-r--r--. 1 root root 0 Nov 22 06:19 memory.failcnt
--w-------. 1 root root 0 Nov 22 06:19 memory.force_empty
-rw-r--r--. 1 root root 0 Nov 22 06:19 memory.kmem.failcnt
-rw-r--r--. 1 root root 0 Nov 22 06:19 memory.kmem.limit_in_bytes
-rw-r--r--. 1 root root 0 Nov 22 06:19 memory.kmem.max_usage_in_bytes
-r--r--r--. 1 root root 0 Nov 22 06:19 memory.kmem.slabinfo
-rw-r--r--. 1 root root 0 Nov 22 06:19 memory.kmem.tcp.failcnt
-rw-r--r--. 1 root root 0 Nov 22 06:19 memory.kmem.tcp.limit_in_bytes
-rw-r--r--. 1 root root 0 Nov 22 06:19 memory.kmem.tcp.max_usage_in_bytes
-r--r--r--. 1 root root 0 Nov 22 06:19 memory.kmem.tcp.usage_in_bytes
-r--r--r--. 1 root root 0 Nov 22 06:19 memory.kmem.usage_in_bytes
-rw-r--r--. 1 root root 0 Nov 22 06:19 memory.limit_in_bytes
-rw-r--r--. 1 root root 0 Nov 22 06:19 memory.max_usage_in_bytes
-rw-r--r--. 1 root root 0 Nov 22 06:19 memory.memsw.failcnt
-rw-r--r--. 1 root root 0 Nov 22 06:19 memory.memsw.limit_in_bytes
-rw-r--r--. 1 root root 0 Nov 22 06:19 memory.memsw.max_usage_in_bytes
-r--r--r--. 1 root root 0 Nov 22 06:19 memory.memsw.usage_in_bytes
-rw-r--r--. 1 root root 0 Nov 22 06:19 memory.move_charge_at_immigrate
-r--r--r--. 1 root root 0 Nov 22 06:19 memory.numa_stat
-rw-r--r--. 1 root root 0 Nov 22 06:19 memory.oom_control
----------. 1 root root 0 Nov 22 06:19 memory.pressure_level
-rw-r--r--. 1 root root 0 Nov 22 06:19 memory.soft_limit_in_bytes
-r--r--r--. 1 root root 0 Nov 22 06:19 memory.stat
-rw-r--r--. 1 root root 0 Nov 22 06:19 memory.swappiness
-r--r--r--. 1 root root 0 Nov 22 06:19 memory.usage_in_bytes
-rw-r--r--. 1 root root 0 Nov 22 06:19 memory.use_hierarchy
-rw-r--r--. 1 root root 0 Nov 22 06:19 notify_on_release
-rw-r--r--. 1 root root 0 Nov 22 06:19 tasks
```

我们来看看是怎么限制的：

```shell
[root@rain-kubernetes-3 ~]# cat /sys/fs/cgroup/cpu/kubepods/pod866d47f4-4d12-41c1-bbfd-51c81517a8fd/0313ba90d47dd7dd2c6c2961aab9d28cb150ec079433db479ea7f266fbee4354/cpu.cfs_period_us
100000     # 100000us->100ms
[root@rain-kubernetes-3 ~]# cat /sys/fs/cgroup/cpu/kubepods/pod866d47f4-4d12-41c1-bbfd-51c81517a8fd/0313ba90d47dd7dd2c6c2961aab9d28cb150ec079433db479ea7f266fbee4354/cpu.cfs_quota_us
70000      # 70000us->70ms->700millicpu

[root@rain-kubernetes-3 ~]# cat /sys/fs/cgroup/cpu/kubepods/pod866d47f4-4d12-41c1-bbfd-51c81517a8fd/0313ba90d47dd7dd2c6c2961aab9d28cb150ec079433db479ea7f266fbee4354/cpu.shares
716 

[root@rain-kubernetes-3 ~]# cat /sys/fs/cgroup/memory/kubepods/pod866d47f4-4d12-41c1-bbfd-51c81517a8fd/0313ba90d47dd7dd2c6c2961aab9d28cb150ec079433db479ea7f266fbee4354/memory.limit_in_bytes
209715200   # 200Mi
```

> cpu.shares用来设置CPU的相对值，并且是针对所有的CPU（内核），默认值是1024，假如系统中有两个cgroup，分别是A和B，A的shares值是1024，B的shares值是512，那么A将获得1024/(1204+512)=66%的CPU资源，而B将获得33%的CPU资源。

## Burstable

而当 Pod 不满足 Guaranteed 的条件，但至少有一个 Container 设置了 requests。那么这个 Pod 就会被划分到 Burstable 类别。

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo-2
  namespace: rain-qos
spec:
  containers:
  - name: qos-demo-2-ctr
    image: nginx
    resources:
      limits:
        memory: "200Mi"
      requests:
        memory: "100Mi"
```

我们刚起的这个pod的container的cgroup位置在burstable的文件夹，只设置了memory的limit：

```shell
[root@rain-kubernetes-1 burstable]# kubectl get pods -n qos-demo-2-ctr -n rain-qos -o wide | grep qos-demo-2
qos-demo-2   1/1     Running   0          38s   10.168.0.2   rain-kubernetes-1   <none>           <none>

[root@rain-kubernetes-1 ~]# docker ps | grep qos-demo-2
1ac54405258b        nginx                              "nginx -g 'daemon of…"   37 minutes ago      Up 37 minutes                           k8s_qos-demo-2-ctr_qos-demo-2_rain-qos_f9a6d832-765f-4048-99f4-cd81ccf35466_0
```

```shell
[root@rain-kubernetes-1 ~]# cat /sys/fs/cgroup/memory/kubepods/burstable/podf9a6d832-765f-4048-99f4-cd81ccf35466/1ac54405258b60d274c6b42f23ad556b70425e442b914a8902aa6c5c331d0e68/memory.limit_in_bytes
209715200 # 200Mi

[root@rain-kubernetes-1 ~]# cat /sys/fs/cgroup/cpu/kubepods/burstable/podf9a6d832-765f-4048-99f4-cd81ccf35466/1ac54405258b60d274c6b42f23ad556b70425e442b914a8902aa6c5c331d0e68/cpu.rt_period_us
1000000

[root@rain-kubernetes-1 ~]# cat /sys/fs/cgroup/cpu/kubepods/burstable/podf9a6d832-765f-4048-99f4-cd81ccf35466/1ac54405258b60d274c6b42f23ad556b70425e442b914a8902aa6c5c331d0e68/cpu.shares
2 
```

设置 Cgroups 限制的时候，kubelet 则会按照 limits 的值来进行设置。

## BestEffort

而如果一个 Pod 既没有设置 requests，也没有设置 limits，那么它的 QoS 类别就是BestEffort。

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo-3
  namespace: rain-qos
spec:
  containers:
  - name: qos-demo-3-ctr
    image: nginx
```

```shell
[root@rain-kubernetes-1 ~]# kubectl get pods -n rain-qos -o wide | grep qos-demo-3
qos-demo-3   1/1     Running   0          105s   10.168.2.3   rain-kubernetes-3   <none>           <none>

[root@rain-kubernetes-3 ~]# docker ps | grep qos-demo-3
28c0f1b6f68d        nginx                              "nginx -g 'daemon of…"   9 minutes ago       Up 9 minutes                            k8s_qos-demo-3-ctr_qos-demo-3_rain-qos_0df2a4c7-9f1d-4b87-84c2-400b1268a9e0_0
```

看看cgroups的配置：

```shell
[root@rain-kubernetes-3 ~]# cat /sys/fs/cgroup/cpu/kubepods/besteffort/pod0df2a4c7-9f1d-4b87-84c2-400b1268a9e0/28c0f1b6f68d02bdeb303c55dec6117923ebc364b4f4bff9fdca562bca616d86/cpu.cfs_quota_us
-1
[root@rain-kubernetes-3 ~]# cat /sys/fs/cgroup/cpu/kubepods/besteffort/pod0df2a4c7-9f1d-4b87-84c2-400b1268a9e0/28c0f1b6f68d02bdeb303c55dec6117923ebc364b4f4bff9fdca562bca616d86/cpu.cfs_period_us
100000

[root@rain-kubernetes-3 ~]# cat /sys/fs/cgroup/cpu/kubepods/besteffort/pod0df2a4c7-9f1d-4b87-84c2-400b1268a9e0/28c0f1b6f68d02bdeb303c55dec6117923ebc364b4f4bff9fdca562bca616d86/cpu.shares
2

[root@rain-kubernetes-3 ~]# cat /sys/fs/cgroup/memory/kubepods/besteffort/pod0df2a4c7-9f1d-4b87-84c2-400b1268a9e0/28c0f1b6f68d02bdeb303c55dec6117923ebc364b4f4bff9fdca562bca616d86/memory.limit_in_bytes
9223372036854771712
```



# Eviction

Eviction Threshold 对应的是 kubernetes 的 eviction policy 特性。该特性也是 kubernetes 引入的用于保护物理节点稳定性的重要特性。具体地说，当 Kubernetes 所管理的宿主机上不可压缩资源短缺时，就有可能触发Eviction。比如，可用内存（memory.available）、可用的宿主机磁盘空间（nodefs.available），以及容
器运行时镜像存储空间（imagefs.available）等等。

可以为压缩资源分别指定一个 eviction hard threshold，即资源量的阈值。

Kubernetes 设置的 Eviction 的默认阈值：

```shell
memory.available<100Mi
nodefs.available<10%
nodefs.inodesFree<5%
imagefs.available<15%
```

Eviction 在 Kubernetes 里其实分为 Soft 和 Hard 两种模式：

* soft eviction
  * Soft Eviction 允许你为 Eviction 过程设置一段“优雅时间”，比如上面例子里的imagefs.available=2m，就意味着当 imagefs 不足的阈值达到 2 分钟之后，kubelet 才会开始Eviction 的过程。
* hard eviction
  * Hard Eviction 模式下，Eviction 过程就会在阈值达到之后立刻开始

> Kubernetes 计算 Eviction 阈值的数据来源，主要依赖于从 Cgroups 读取到的值，以及使用 cAdvisor 监控到的数据。

当宿主机的 Eviction 阈值达到后，就会进入 MemoryPressure 或者 DiskPressure 状态，从而避免新的 Pod 被调度到这台宿主机上。

而当 Eviction 发生的时候，kubelet 具体会挑选哪些 Pod 进行删除操作，会以下面这个顺序进行删除：

1. besteffort的pod
2. burstable，并且发生“饥饿”的资源使用量已经超出了 requests 的Pod
3. guaranteed，Kubernetes 会保证只有当 Guaranteed 类别的 Pod的资源使用量超过了其 limits 的限制，或者宿主机本身正处于 Memory Pressure 状态时，Guaranteed 的 Pod 才可能被选中进行 Eviction 操作。