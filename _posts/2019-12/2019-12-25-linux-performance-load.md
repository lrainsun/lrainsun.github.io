---
layout: post
title:  "平均负载"
date:   2019-12-25 9:55:00 +0800
categories: Linux
tags: Linux-Performance
excerpt: 平均负载
mathjax: true
typora-root-url: ../
---

# 平均负载的概念

平均负载是指单位时间内，系统处于*可运行状态*和*不可中断状态*的平均进程数，也就是*平均活跃进程数*，它和 CPU 使用率并没有直接关系。

* 可运行状态的进程：指正在使用 CPU 或者正在等待 CPU 的进程，也就是我们常用ps 命令看到的，处于 R 状态（Running 或 Runnable）的进程。
* 不可中断的进程：处于内核态关键流程中的进程，并且这些流程是不可打断的，比如最常见的是等待硬件设备的 I/O 响应，也就是我们在 ps 命令中看到的 D 状态（Uninterruptible Sleep，也称为 Disk Sleep）的进程。不可中断状态实际上是系统对进程和硬件设备的一种保护机制。

所以简单理解：平均负载其实就是平均活跃进程数

实际计算：活跃进程数的指数衰减平均值（系统的更快速的计算方法）

## 举个例子

当平均负载为 2 时：

* 在只有 2 个 CPU 的系统上，意味着所有的 CPU 都刚好被完全占用
* 在 4 个 CPU 的系统上，意味着 CPU 有 50% 的空闲
* 而在只有 1 个 CPU 的系统中，则意味着有一半的进程竞争不到 CPU

## 平均负载多少合理

平均负载最理想的情况，是每个cpu上都刚好有一个运行着的进程，也就是cpu个数。有了cpu个数，我们就可以判断，当平均负载比cpu个数还大的时候，系统已经出现了过载。

所以要先知道系统有几个cpu，cpu个数可以通过如下方式获取：

```shell
[root@rain-kubernetes-1 ~]# grep "model name" -rn /proc/cpuinfo | wc -l
4
```

那么，再来获取一下平均负载

```shell
[root@rain-kubernetes-1 ~]# uptime
 00:41:10 up 6 days, 18:40,  1 user,  load average: 0.12, 0.10, 0.09
```

uptime命令：

* 00：41：10 //当前时间
* up 6 days, 18:40 //系统运行的时间
* 1 user //正在登录用户数
* load average: 0.12, 0.10, 0.09 //过去1分钟平均负载，过去5分钟平均负载，过去15分钟平均负载

### 三个平均负载的解读

* 如果 1 分钟、5 分钟、15 分钟的三个值基本相同，或者相差不大，那就说明系统负载很平稳
* 如果 1 分钟的值远小于 15 分钟的值，就说明系统最近 1 分钟的负载在减少，而过去15 分钟内却有很大的负载
* 如果 1 分钟的值远大于 15 分钟的值，就说明最近 1 分钟的负载在增加，这种增加有可能只是临时性的，也有可能还会持续增加下去，所以就需要持续观察
* 一旦 1分钟的平均负载接近或超过了 CPU 的个数，就意味着系统正在发生过载的问题，这时就
  得分析调查是哪里导致的问题，并要想办法优化了

> 当平均负载高于 CPU 数量 70%（并不绝对） 的时候，你就应该分析排查负载高的问题了。一旦负载过高，就可能导致进程响应变慢，进而影响服务的正常功能。最推荐的方法，还是把系统的平均负载监控起来，然后根据更多的历史数据，判断负载的变化趋势。当发现负载有明显升高趋势时，比如说负载翻倍了，你再去做分析和调查

# 平均负载与CPU使用率

平均负载是指单位时间内，处于可运行状态和不可中断状态的进程数。所以，它不仅包括了正在使用 CPU 的进程，还包括等待 CPU 和等待I/O 的进程。

CPU 使用率，是单位时间内 CPU 繁忙情况的统计，跟平均负载并不一定完全对应。

* CPU 密集型进程，使用大量 CPU 会导致平均负载升高，此时这两者是一致的；
* I/O 密集型进程，等待 I/O 也会导致平均负载升高，但 CPU 使用率不一定很高；
* 大量等待 CPU 的进程调度也会导致平均负载升高，此时的 CPU 使用率也会比较高。

# 实验

## 准备

需要用到stress和sysstat：

* stress：stress 是一个 Linux 系统压力测试工具，这里我们用作异常进程模拟平均负载升高的场景

* sysstat：sysstat 包含了常用的 Linux 性能工具，用来监控和分析系统的性能。我们需要用到两个命令：
  * mpstat：常用的多核 CPU 性能分析工具，用来实时查看每个 CPU 的性能指标，以及所有 CPU 的平均指标
  * pidstat：是一个常用的进程性能分析工具，用来实时查看进程的 CPU、内存、I/O 以及上下文切换等性能指标

安装stress & sysstat

```shell
yum install -y epel-release
yum install -y stress
yum install -y sysstat
```

## CPU密集型进程

开启三个终端：

第一个终端运行stress命令，模拟一个CPU 使用率 100% 的场景：

```shell
[root@rain-kubernetes-1 ~]# stress --cpu 1 --timeout 600
stress: info: [29642] dispatching hogs: 1 cpu, 0 io, 0 vm, 0 hdd
```

第二个终端运行uptime查看平均负载的变化情况，可以看到一分钟的负载从0.09会缓慢增加到1左右

```shell
 watch -d uptime  //-d 参数表示高亮显示变化的区域
 
 01:31:25 up 6 days, 19:30,  3 users,  load average: 0.09, 0.53, 0.43
 
 01:33:23 up 6 days, 19:32,  3 users,  load average: 1.04, 0.74, 0.52
```

第三个终端运行 mpstat 查看 CPU 使用率的变化情况

可以看到cpu 2的CPU使用率是100%，但它的iowait只有0。这说明，平均负载的升高正是由于 CPU 使用率为 100%

```shell
mpstat -P ALL 5 //-P ALL 表示监控所有 CPU，后面数字 5 表示间隔 5 秒后输出一组数据
Linux 3.10.0-693.11.1.el7.x86_64 (rain-kubernetes-1.localdomain) 	12/25/2019 	_x86_64_	(4 CPU)

01:33:23 AM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
01:33:28 AM  all   26.49    0.00    0.60    0.05    0.00    0.05    0.00    0.00    0.00   72.80
01:33:28 AM    0    2.00    0.00    1.00    0.00    0.00    0.00    0.00    0.00    0.00   96.99
01:33:28 AM    1    1.81    0.00    0.80    0.00    0.00    0.00    0.00    0.00    0.00   97.38
01:33:28 AM    2  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
01:33:28 AM    3    2.22    0.00    0.60    0.00    0.00    0.00    0.00    0.00    0.00   97.18
```

可以通过pidstat命令来查看哪个进程导致了CPU使用率100%

```shell
[root@rain-kubernetes-1 ~]# pidstat -u 5 1 //间隔5秒后输出一组数据
Linux 3.10.0-693.11.1.el7.x86_64 (rain-kubernetes-1.localdomain) 	12/25/2019 	_x86_64_	(4 CPU)

01:39:48 AM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
01:39:53 AM     0         9    0.00    0.20    0.00    0.20     1  rcu_sched
01:39:53 AM     0       354    0.20    0.00    0.00    0.20     2  systemd-journal
01:39:53 AM     0       518    1.40    0.40    0.00    1.80     1  kubelet
01:39:53 AM     0       894    1.00    0.00    0.00    1.00     2  dockerd
01:39:53 AM     0      3411    0.40    0.20    0.00    0.60     0  etcd
01:39:53 AM     0      3603    0.60    1.00    0.00    1.60     3  kube-apiserver
01:39:53 AM     0      3808    1.60    0.20    0.00    1.80     2  kube-controller
01:39:53 AM     0     31754  100.00    0.00    0.00  100.00     3  stress
01:39:53 AM     0     31867    0.00    0.20    0.00    0.20     0  pidstat

Average:      UID       PID    %usr %system  %guest    %CPU   CPU  Command
Average:        0         9    0.00    0.20    0.00    0.20     -  rcu_sched
Average:        0       354    0.20    0.00    0.00    0.20     -  systemd-journal
Average:        0       518    1.40    0.40    0.00    1.80     -  kubelet
Average:        0       894    1.00    0.00    0.00    1.00     -  dockerd
Average:        0      3411    0.40    0.20    0.00    0.60     -  etcd
Average:        0      3603    0.60    1.00    0.00    1.60     -  kube-apiserver
Average:        0      3808    1.60    0.20    0.00    1.80     -  kube-controller
Average:        0     31754  100.00    0.00    0.00  100.00     -  stress
Average:        0     31867    0.00    0.20    0.00    0.20     -  pidstat
```

可以看到stress进程的CPU使用率为100%

## I/O密集型进程

这次终端一运行stress命令，模拟I/O 压力，即不停地执行 sync

```shell
[root@rain-kubernetes-1 ~]# stress -i 1 --timeout 600
stress: info: [452] dispatching hogs: 0 cpu, 1 io, 0 vm, 0 hdd
```

终端二还是运行uptime查看平均负载，1分钟负载还是缓慢增加到1左右

```
[root@rain-kubernetes-1 ~]# watch -d uptime

 01:47:42 up 6 days, 19:46,  3 users,  load average: 1.05, 1.02, 0.77
```

终端三运行mpstat查看CPU使用率的变化情况，CPU 2的sys(内核态的CPU)利用率高达98%，iowait 0.4%。看起来与我们想要的结果不符（我们想要制造I/O压力）。所以此处有疑问，然后google一下：

> -i,--io：表示调用sync()，它表示通过系统调用 sync() 来模拟 I/O 的问题；
>
>  但这种方法实际上并不可靠，因为 sync() 的本意是刷新内存缓冲区的数据到磁盘中，以确保同步。
>
>  如果缓冲区内本来就没多少数据，那读写到磁盘中的数据也就不多，也就没法产生 I/O 压力。
>
>  这一点，在使用 SSD 磁盘的环境中尤为明显，很可能你的 iowait 总是 0，却单纯因为大量的系统调用，导致了系统CPU使用率 sys 升高。

看起来正是我们这种情况，所以在这里使用stress并不能达到我们想要的效果。可以换用stress-ng

```
yum install -y stress-ng
```

stess-ng是stress的下一代，功能更加完善

```
[root@rain-kubernetes-1 ~]# mpstat -P ALL 5
Linux 3.10.0-693.11.1.el7.x86_64 (rain-kubernetes-1.localdomain) 	12/25/2019 	_x86_64_	(4 CPU)
01:46:09 AM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
01:46:14 AM  all    1.51    0.00   25.40    0.10    0.00    0.00    0.00    0.00    0.00   72.99
01:46:14 AM    0    2.21    0.00    1.01    0.00    0.00    0.00    0.00    0.00    0.00   96.78
01:46:14 AM    1    1.61    0.00    1.00    0.20    0.00    0.20    0.00    0.00    0.00   96.99
01:46:14 AM    2    0.00    0.00   98.20    0.40    0.00    0.00    0.00    0.00    0.00    1.40
01:46:14 AM    3    2.00    0.00    1.00    0.20    0.00    0.00    0.00    0.00    0.00   96.79
```

这样，终端一运行stress-ng制造I/O压力

```shell
[root@rain-kubernetes-1 ~]# stress-ng -i 1 --hdd 4 --timeout 600 //--hdd start N workers spinning on write()/unlink() -i start N workers spinning on sync()
stress-ng: info:  [15908] dispatching hogs: 4 hdd, 1 io
```

终端二uptime，平均负载已超过cpu个数4

```shell
[root@rain-kubernetes-1 ~]# watch -d uptime
 02:36:37 up 6 days, 20:35,  3 users,  load average: 6.48, 5.10, 3.52
```

终端三，这次可以看到，iowait的比例很高，说明平均负载的升高时因为iowait的升高，一直在等待io处理，进程频繁进行IO操作，导致平均负载高而CPU使用率不那么高

```shell
[root@rain-kubernetes-1 ~]# mpstat -P ALL 5
Linux 3.10.0-693.11.1.el7.x86_64 (rain-kubernetes-1.localdomain) 	12/25/2019 	_x86_64_	(4 CPU)

02:37:14 AM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
02:37:19 AM  all    1.59    0.00   35.27   56.78    0.00    0.15    0.05    0.00    0.00    6.16
02:37:19 AM    0    1.03    0.00   50.62   43.42    0.00    0.00    0.21    0.00    0.00    4.73
02:37:19 AM    1    1.64    0.00   27.66   61.89    0.00    0.41    0.00    0.00    0.00    8.40
02:37:19 AM    2    1.84    0.00   31.56   60.45    0.00    0.00    0.00    0.00    0.00    6.15
02:37:19 AM    3    1.85    0.00   31.07   61.52    0.00    0.00    0.00    0.00    0.00    5.56
```

## 大量进程的场景

当系统中运行进程超出 CPU 运行能力时，就会出现等待 CPU 的进程。

终端一模拟16个进程（我们只有4个CPU）

```shell
[root@rain-kubernetes-1 ~]# stress-ng -c 16 --timeout 600
stress-ng: info:  [32617] dispatching hogs: 16 cpu
```

 终端二uptime

```shell
[root@rain-kubernetes-1 ~]# watch -d uptime
 03:33:24 up 6 days, 21:32,  3 users,  load average: 16.49, 8.92, 4.53
```

终端三，可以看到每个CPU使用率并不高，是因为16个进程在抢4个cpu，每个进程等待CPU的时间比较多，而这些些超出 CPU 计算能力的进程，最终导致 CPU 过载。

```shell
[root@rain-kubernetes-1 ~]# pidstat -u 5 1 | grep -E 'stress|Command'
03:31:54 AM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
03:31:59 AM     0     32618   26.24    0.00    0.00   26.24     0  stress-ng-cpu
03:31:59 AM     0     32619   24.65    0.00    0.00   24.65     1  stress-ng-cpu
03:31:59 AM     0     32620   22.07    0.20    0.00   22.27     3  stress-ng-cpu
03:31:59 AM     0     32621   26.64    0.00    0.00   26.64     0  stress-ng-cpu
03:31:59 AM     0     32622   25.25    0.00    0.00   25.25     3  stress-ng-cpu
03:31:59 AM     0     32623   22.66    0.00    0.00   22.66     2  stress-ng-cpu
03:31:59 AM     0     32624   22.07    0.20    0.00   22.27     3  stress-ng-cpu
03:31:59 AM     0     32625   24.25    0.00    0.00   24.25     0  stress-ng-cpu
03:31:59 AM     0     32626   22.27    0.00    0.00   22.27     3  stress-ng-cpu
03:31:59 AM     0     32627   23.86    0.00    0.00   23.86     2  stress-ng-cpu
03:31:59 AM     0     32628   24.45    0.00    0.00   24.45     0  stress-ng-cpu
03:31:59 AM     0     32629   24.06    0.00    0.00   24.06     2  stress-ng-cpu
03:31:59 AM     0     32630   25.05    0.00    0.00   25.05     3  stress-ng-cpu
03:31:59 AM     0     32631   24.25    0.00    0.00   24.25     2  stress-ng-cpu
03:31:59 AM     0     32632   25.25    0.20    0.00   25.45     1  stress-ng-cpu
03:31:59 AM     0     32633   27.24    0.00    0.00   27.24     0  stress-ng-cpu
Average:      UID       PID    %usr %system  %guest    %CPU   CPU  Command
Average:        0     32618   26.24    0.00    0.00   26.24     -  stress-ng-cpu
Average:        0     32619   24.65    0.00    0.00   24.65     -  stress-ng-cpu
Average:        0     32620   22.07    0.20    0.00   22.27     -  stress-ng-cpu
Average:        0     32621   26.64    0.00    0.00   26.64     -  stress-ng-cpu
Average:        0     32622   25.25    0.00    0.00   25.25     -  stress-ng-cpu
Average:        0     32623   22.66    0.00    0.00   22.66     -  stress-ng-cpu
Average:        0     32624   22.07    0.20    0.00   22.27     -  stress-ng-cpu
Average:        0     32625   24.25    0.00    0.00   24.25     -  stress-ng-cpu
Average:        0     32626   22.27    0.00    0.00   22.27     -  stress-ng-cpu
Average:        0     32627   23.86    0.00    0.00   23.86     -  stress-ng-cpu
Average:        0     32628   24.45    0.00    0.00   24.45     -  stress-ng-cpu
Average:        0     32629   24.06    0.00    0.00   24.06     -  stress-ng-cpu
Average:        0     32630   25.05    0.00    0.00   25.05     -  stress-ng-cpu
Average:        0     32631   24.25    0.00    0.00   24.25     -  stress-ng-cpu
Average:        0     32632   25.25    0.20    0.00   25.45     -  stress-ng-cpu
Average:        0     32633   27.24    0.00    0.00   27.24     -  stress-ng-cpu
```

# 总结

平均负载提供了一个快速查看系统整体性能的手段，反映了整体的负载情况。但只看平均负载本身，我们并不能直接发现，到底是哪里出现了瓶颈。

* 平均负载高有可能是 CPU 密集型进程导致的；
* 平均负载高并不一定代表 CPU 使用率高，还有可能是 I/O 更繁忙了；
* 当发现负载高的时候，你可以使用 mpstat、pidstat 等工具，辅助分析负载的来源。

