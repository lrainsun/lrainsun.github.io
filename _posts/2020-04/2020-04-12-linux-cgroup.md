---
layout: post
title:  "CGroups"
date:   2020-04-12 23:00:00 +0800
categories: Linux
tags: Linux-Service
excerpt: CGroups
mathjax: true
typora-root-url: ../
---

# Cgroups简介

**Cgroups**：是linux内核中的一个机制。是control groups的缩写，是Linux内核提供的一种可以限制、记录、隔离进程组（process groups）所使用的物理资源（如：cpu,memory,IO等等）的机制。

**CGroup**：是控制组，是cgroups实现资源控制的一个基本单位。 是将任意进程进行分组化管理的 Linux 内核功能。CGroup 本身是提供将进程进行分组化管理的功能和接口的基础结构，I/O 或内存的分配控制等具体的资源管理功能是通过这个功能来实现的。这些具体的资源管理功能称为 CGroup 子系统或控制器。CGroup 子系统有控制内存的 Memory 控制器、控制进程调度的 CPU 控制器等。运行中的内核可以使用的 Cgroup 子系统由/proc/cgroup 来确认。

# 功能

Cgroups提供了以下功能：

* 限制进程组可以使用的资源数量（Resource limiting ）。比如：memory子系统可以为进程组设定一个memory使用上限，一旦进程组使用的内存达到限额再申请内存，就会出发OOM（out of memory）。
* 进程组的优先级控制（Prioritization ）。比如：可以使用cpu子系统为某个进程组分配特定cpu share。
* 记录进程组使用的资源数量（Accounting ）。比如：可以使用cpuacct子系统记录某个进程组使用的cpu时间
* 进程组隔离（Isolation）。比如：使用ns子系统可以使不同的进程组使用不同的namespace，以达到隔离的目的，不同的进程组有各自的进程、网络、文件系统挂载空间。
* 进程组控制（Control）。比如：使用freezer子系统可以将进程组挂起和恢复。

# 概念

cgroup 是 Linux 下的一种将进程按组进行管理的机制，在用户层看来，cgroup 技术就是把系统中的所有进程组织成一颗一颗独立的树，每棵树都包含系统的所有进程，树的每个节点是一个进程组，而每颗树又和一个或者多个 `subsystem` 关联，树的作用是将进程分组，而 `subsystem` 的作用就是对这些组进行操作。cgroup 主要包括：

* **task**：任务表示系统的一个进程或线程。

- **subsystem** : 一个 subsystem 就是一个内核模块，他被关联到一颗 cgroup 树之后，就会在树的每个节点（进程组）上做具体的操作。一个子系统是一个资源调度控制器（比如cpu, memory, blkio等等都是子系统）。
- **hierarchy** : 层级，一个 `hierarchy` 可以理解为一棵 cgroup 树，树的每个节点就是一个进程组，每棵树都会与零到多个 `subsystem` 关联。在一颗树里面，会包含 Linux 系统中的所有进程，但每个进程只能属于一个节点（进程组）。系统中可以有很多颗 cgroup 树，每棵树都和不同的 subsystem 关联，一个进程可以属于多颗树，即一个进程可以属于多个进程组，只是这些进程组和不同的 subsystem 关联。目前 Linux 支持 12 种 subsystem，如果不考虑不与任何 subsystem 关联的情况（systemd 就属于这种情况），Linux 里面最多可以建 12 颗 cgroup 树，每棵树关联一个 subsystem，当然也可以只建一棵树，然后让这棵树关联所有的 subsystem。当一颗 cgroup 树不和任何 subsystem 关联的时候，意味着这棵树只是将进程进行分组，至于要在分组的基础上做些什么，将由应用程序自己决定，`systemd` 就是一个这样的例子。

![图 1. CGroup 层级图](/../assets/images/img001.png)

CPU 和 Memory 两个子系统有自己独立的层级系统，而又通过 Task Group 取得关联关系。

1. 每次在系统中创建新层级时，该系统中的所有任务都是那个层级的默认 cgroup（我们称之为 root cgroup，此 cgroup 在创建层级时自动创建，后面在该层级中创建的 cgroup 都是此 cgroup 的后代）的初始成员；
2. 一个子系统最多只能附加到一个层级；
3. 一个层级可以附加多个子系统；
4. 一个任务可以是多个 cgroup 的成员，但是这些 cgroup 必须在不同的层级；
5. 系统中的进程（任务）创建子进程（任务）时，该子任务自动成为其父进程所在 cgroup 的成员。然后可根据需要将该子任务移动到不同的 cgroup 中，但开始时它总是继承其父任务的 cgroup。

# cgroups长啥样

在`/sys/fs/cgroup`目录下

```shell
[root@rain-working-vm ~]# ls -al /sys/fs/cgroup
total 0
drwxr-xr-x. 13 root root 340 Mar 26 08:39 .
drwxr-xr-x.  6 root root   0 Mar 26 08:39 ..
drwxr-xr-x.  5 root root   0 Mar 26 08:39 blkio  —— 块设备设定输入/输出限制，比如物理设备（磁盘，固态硬盘，USB 等等）
lrwxrwxrwx.  1 root root  11 Mar 26 08:39 cpu -> cpu,cpuacct —— 使用调度程序提供对 CPU 的 cgroup 任务访问
lrwxrwxrwx.  1 root root  11 Mar 26 08:39 cpuacct -> cpu,cpuacct —— 自动生成 cgroup 中任务所使用的 CPU 报告
drwxr-xr-x.  5 root root   0 Mar 26 08:39 cpu,cpuacct
drwxr-xr-x.  3 root root   0 Mar 26 08:39 cpuset —— 为 cgroup 中的任务分配独立 CPU（在多核系统）和内存节点
drwxr-xr-x.  5 root root   0 Mar 26 08:39 devices —— 可允许或者拒绝 cgroup 中的任务访问设备
drwxr-xr-x.  3 root root   0 Mar 26 08:39 freezer —— 挂起或者恢复 cgroup 中的任务
drwxr-xr-x.  3 root root   0 Mar 26 08:39 hugetlb
drwxr-xr-x.  5 root root   0 Mar 26 08:39 memory
lrwxrwxrwx.  1 root root  16 Mar 26 08:39 net_cls -> net_cls,net_prio —— 使用等级识别符（classid）标记网络数据包，可允许 Linux 流量控制程序（tc）识别从具体 cgroup 中生成的数据包
drwxr-xr-x.  3 root root   0 Mar 26 08:39 net_cls,net_prio
lrwxrwxrwx.  1 root root  16 Mar 26 08:39 net_prio -> net_cls,net_prio
drwxr-xr-x.  3 root root   0 Mar 26 08:39 perf_event
drwxr-xr-x.  3 root root   0 Mar 26 08:39 pids
drwxr-xr-x.  5 root root   0 Mar 26 08:39 systemd
```

在cpu子系统的层级下：

```shell
[root@rain-working-vm ~]# ls  /sys/fs/cgroup/cpu/
cgroup.clone_children  cgroup.procs          cpuacct.stat   cpuacct.usage_percpu  cpu.cfs_quota_us  cpu.rt_runtime_us  cpu.stat  notify_on_release  system.slice  user.slice
cgroup.event_control   cgroup.sane_behavior  cpuacct.usage  cpu.cfs_period_us     cpu.rt_period_us  cpu.shares         docker    release_agent      tasks
```

文件名前缀为cgroup的以及没有前缀的文件是由cgroup的基础结构提供的特殊文件：

| 文件名               | R/W  | 用途                                                         |
| :------------------- | :--- | :----------------------------------------------------------- |
| Release_agent        | RW   | 删除分组时执行的命令，这个文件只存在于根分组                 |
| Notify_on_release    | RW   | 设置是否执行 release_agent。为 1 时执行                      |
| Tasks                | RW   | 属于分组的线程 TID 列表                                      |
| Cgroup.procs         | R    | 属于分组的进程 PID 列表。仅包括多线程进程的线程 leader 的 TID，这点与 tasks 不同 |
| Cgroup.event_control | RW   | 监视状态变化和分组删除事件的配置文件                         |

在cpu子系统下创建一个目录，这个目录就是一个新创建的cgroup控制组

```shell
[root@rain-working-vm cpu]# mkdir rain_cgroup1
[root@rain-working-vm cpu]# ls rain_cgroup1/
cgroup.clone_children  cgroup.procs  cpuacct.usage         cpu.cfs_period_us  cpu.rt_period_us   cpu.shares  notify_on_release
cgroup.event_control   cpuacct.stat  cpuacct.usage_percpu  cpu.cfs_quota_us   cpu.rt_runtime_us  cpu.stat    tasks
```

一旦目录被创建，就会自动生成这么多文件，rain_cgroup1继承了根cgroup的配置属性。

系统中运行的所有线程的TID都包含在`/sys/fs/cgroup/cpu/tasks`中，表示全部线程都属于这个（根）分组。

而我们刚刚创建的目录是一个子目录，其中的task是空的，我们可以把线程TID放进去。

