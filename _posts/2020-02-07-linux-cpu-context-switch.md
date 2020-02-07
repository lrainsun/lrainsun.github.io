---
layout: post
title:  "CPU上下文切换分析手段"
date:   2020-02-07 20:00:00 +0800
categories: Linux
tags: Linux-Performance
excerpt: CPU上下文切换分析手段
mathjax: true
typora-root-url: ../
---

# CPU上下文切换

回顾一下，我们是从研究系统平均负载开始的：

* 系统平均负载跟CPU使用率没有直接的关系（是单位时间内，系统处于可运行状态和不可中断状态的平均进程数，即活跃进程数）
* 平均负载反映了系统整体的性能
* CPU密集型进程有可能导致平均负载变高
* I/O繁忙进程也有可能导致平均负载变高
* <u>进程在等待CPU的时候也会导致平均负载变高</u>

而之所以进程等待CPU也会让平均负载变高的原因就是因为**CPU上下文切换**

CPU上下文切换可以分为：

* 系统调用（用户态到内核态的转变会有CPU上下文切换，不涉及虚拟内存等进程用户态资源）

* 进程上下文切换（包括了虚拟内存、栈、全局变量等用户空间资源，还包括内核堆栈、寄存器等内核空间状态）
* 线程上下文切换（属于不同进程的线程上下文切换跟进程上下文切换一样，属于同一进程的线程上下文切换可以保持虚拟内存这些资源不动，只切换线程私有数据、寄存器等不共享资源）
* 中断上下文切换（不涉及进程的用户态，只包括内核态中断服务程序执行所必须的状态，包括CPU寄存器、内核堆栈，硬件中断参数等）

# 查看系统CPU上下文切换

## vmstat —— 系统总体上下文切换情况

```shell
[root@rain-kubernetes-1 ~]# vmstat
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 1978064   2088 1158284    0    0    22    14  749  100  1  1 98  0  0
```

cs（context switch）：每秒上下文切换的次数。

in（interrupt）：每秒中断的次数。

r（Running or Runnable）：就绪队列的长度，正在运行和等待 CPU 的进程数。

b（Blocked）：处于不可中断睡眠状态的进程数。

## 进程上下文切换情况

每隔5s输出一组数据

```shell
[root@rain-kubernetes-1 ~]# pidstat -w 5
Linux 3.10.0-693.11.1.el7.x86_64 (rain-kubernetes-1.localdomain) 	02/07/2020 	_x86_64_	(4 CPU)
11:06:34 AM   UID       PID   cswch/s nvcswch/s  Command
11:06:39 AM     0         3      0.40      0.00  ksoftirqd/0
11:06:39 AM     0         6      0.20      0.00  kworker/u8:0
```

cswch：每秒自愿上下文切换（voluntary context switches）的次数。自愿上下文切换，是指进程无法获取所需资源，导致的上下文切换。比如说，I/O、内存等系统资源不足时，就会发生自愿上下文切换。

nvcswch：每秒非自愿上下文切换（non voluntary context switches）的次数。非自愿上下文切换，则是指进程由于时间片已到等原因，被系统强制调度，进而发生的上下文切换。比如说，大量进程都在争抢 CPU 时，就容易发生非自愿上下文切换。

# 模拟多线程切换

以10个进程运行5分钟的基准测试，模拟多线程切换的问题

```shell
[root@rain-kubernetes-1 ~]# sysbench --threads=10 --time=300 threads run
sysbench 1.0.17 (using system LuaJIT 2.0.4)
```

## 查看上下文切换情况

每隔1秒输出一组数据

```shell
[root@rain-kubernetes-1 ~]# vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 8  0      0 1773640   2088 1332992    0    0     7    12   67    6  1  1 98  0  0
 6  0      0 1773596   2088 1332996    0    0     0    53 150814 1496404 10 73 17  0  0
 9  0      0 1773596   2088 1332996    0    0     0   128 161821 1516791  5 78 17  0  0
```

上下文切换次数（cs）从6增加到1516791

就绪队列（r）增加到9，超过了系统CPU的个数4，所以肯定会有大量CPU竞争

```shell
[root@rain-kubernetes-1 ~]# grep "model name" -rn /proc/cpuinfo | wc -l
4
```

us（user）和sy（system）这两列CPU 使用率加起来达到83%，系统CPU使用率78%，说明CPU主要是被内核占用了

中断次数（in）达到161821，说明中断处理也很多

综合这几个指标，系统的就绪队列过长，正在运行和等待 CPU 的进程数过多，导致了大量的上下文切换，而上下文切换又导致了系统 CPU 的占用率升高。

## 问题根源

```shell
[root@rain-kubernetes-1 ~]# pidstat -w -u 1
Linux 3.10.0-693.11.1.el7.x86_64 (rain-kubernetes-1.localdomain) 	02/07/2020 	_x86_64_	(4 CPU)

11:24:30 AM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
11:24:31 AM     0       538    0.98    0.98    0.00    1.96     2  kubelet
11:24:31 AM     0       928    0.00    0.98    0.00    0.98     3  dockerd
11:24:31 AM     0      3366    0.98    0.98    0.00    1.96     0  kube-controller
11:24:31 AM     0      3853    0.98    0.00    0.00    0.98     3  kube-apiserver
11:24:31 AM     0     29649   25.49  100.00    0.00  100.00     2  sysbench
11:24:31 AM     0     29848    0.00    0.98    0.00    0.98     3  pidstat
11:24:31 AM     0     29874    0.98    0.00    0.00    0.98     0  calico

11:24:30 AM   UID       PID   cswch/s nvcswch/s  Command
11:24:31 AM     0         3     11.76      0.00  ksoftirqd/0
11:24:31 AM     0         7     10.78      0.00  migration/0
11:24:31 AM     0         9    105.88      0.00  rcu_sched
11:24:31 AM     0        13      2.94      0.00  ksoftirqd/1
11:24:31 AM     0        17      1.96      0.00  migration/2
11:24:31 AM     0        18      3.92      0.00  ksoftirqd/2
11:24:31 AM     0        22      5.88      0.00  migration/3
11:24:31 AM     0        23      1.96      0.00  ksoftirqd/3
11:24:31 AM     0       183      9.80      0.00  kauditd
11:24:31 AM     0       284     20.59      0.00  xfsaild/vda1
11:24:31 AM     0       285      3.92      0.00  kworker/2:1H
11:24:31 AM     0       356     13.73      0.00  systemd-journal
11:24:31 AM     0       490     13.73      0.00  auditd
11:24:31 AM     0       531      3.92      0.00  kworker/3:1H
11:24:31 AM    38       540      0.98      0.00  ntpd
11:24:31 AM     0      1092      0.98      0.00  .vasd
11:24:31 AM  5003      1233      0.98      0.00  slimdm
11:24:31 AM     0      1242      2.94      0.00  kworker/1:1H
11:24:31 AM     0      3366     69.61      0.00  kube-controller
11:24:31 AM     0      3682      2.94      0.00  kworker/2:3
11:24:31 AM     0     11533      0.98      0.00  kworker/0:0
11:24:31 AM     0     26105      6.86      0.00  kworker/1:0
11:24:31 AM     0     28408      2.94      0.00  kworker/3:0
11:24:31 AM     0     29848      0.98      0.00  pidstat
11:24:31 AM     0     29874     11.76      8.82  calico
```

sysbench的CPU使用率达到了100%  ——> 导致了系统CPU使用率增高

上下文切换则是来自其他进程，包括非自愿上下文切换频率最高的calico，自愿上下文切换频率最高的rcu_sched

这个切换次数为什么跟上面切换次数对应不起来呢？因为这里是进程的上下文切换次数，要查看线程上下文切换指标要加-t

```shell
[root@rain-kubernetes-1 ~]# pidstat -wt 5
Linux 3.10.0-693.11.1.el7.x86_64 (rain-kubernetes-1.localdomain) 	02/07/2020 	_x86_64_	(4 CPU)
```

要查看发生了哪种中断，可以从/proc/interrupts读。/proc 实际上是 Linux 的一个虚拟文件系统，用于内核空间与用户空间之间的通信。/proc/interrupts 就是这种通信机制的一部分，提供了一个只读的中断使用情况。

```shell
[root@rain-kubernetes-1 ~]# watch -d cat /proc/interrupts
```

![image-20200207195123599](/assets/images/image-20200207195123599.png)

变化速度最快的是重调度中断（RES，rescheduling interrupts），这个中断类型表示，唤醒空闲状态的 CPU 来调度新的任务运行。所以，这里的中断升高还是因为过多任务的调度问题，跟前面上下文切换次数的分析结果是一致的。

# 总结

每秒上下文切换次数多少正常取决于于系统本身的 CPU 性能。在我看来，如果系统的上下文切换次数比较稳定，那么从数百到一万以内，都应该算是正常的。但当上下文切换次数超过一万次，或者切换次数出现数量级的增长时，就很可能已经出现了性能问题。

* 自愿上下文切换变多了，说明进程都在等待资源，有可能发生了 I/O 等其他问题；
* 非自愿上下文切换变多了，说明进程都在被强制调度，也就是都在争抢 CPU，说明 CPU的确成了瓶颈；
* 中断次数变多了，说明 CPU 被中断处理程序占用，还需要通过查看 /proc/interrupts文件来分析具体的中断类型。