---
layout: post
title:  "cpu contention"
date:   2020-04-01 23:00:00 +0800
categories: Linux
tags: Linux-Service
excerpt: cpu contention
mathjax: true
typora-root-url: ../
---

# cpu contention

CPU contention is an event wherein individual CPU components and machines in a virtualized hardware system wait too long for their turn at processing. In such a system, resources (e.g., CPU, memory, etc.) are distributed between different virtual machines (VMs). As different processing resources are assigned to different machines, the schedulers in the system order the input/output and other tasks. The processing of these tasks is delayed when the machines assigned to them are experiencing a CPU contention.

# stolen/steal time

Steal time is the percentage of time a virtual CPU waits for a real CPU while the hypervisor is servicing another virtual processor. for example if steal time=50% means that vcpu can get 50% pcpu time, steal time=100% means that vcpu can not get any pcpu time. 

Steal time is reported in the CPU time fields in `/proc/stat`. It is automatically reported by utilities such as `top` and `vmstat`. It is displayed as "%st", or in the "st" column. Note that it cannot be switched off.

你的虚拟机（VM）会与虚拟环境的宿主机上的多个虚拟机实例共享物理资源。其中之一共享的就是CPU时间切片。如果你的VM的物理机虚拟比是1/4， 那么它的CPU使用率不会限制于25%的CPU时间切片－它能够使用超过它设置的虚拟比。（有别于内存的使用，内存大小是严格控制的）

有这样一个例子：

如果我们把 CPU steal time 性能指标 类比成 售票的过程， 那么过程就是如下：

**0% Steal Time** - 现在是礼拜三下午场：售票口正在工作，先处理第一条队伍的电影观众，然后处理第二条，然后第一条，然后第二条，轮流进行。处理的很快，且没有人在等待。

**50% Steal Time** - 现在是礼拜五晚上： 在队伍中的一个人有一半的时间需要等待另一个在售票口的人完成卖票，而不能立刻买到票。卖票的时间更长了。

**100% Steal Time** - 现在是礼拜五晚上并且 现金出纳金 坏了：所有人都在等待。

## Steal Time远高于0的原因

这里有两种可能性：

你需要一个额定更多CPU资源的虚拟机（你的虚拟机**是**问题）

物理机已经超卖了并且多个虚拟机之间在激烈的竞争资源（你的虚拟机**不是**问题）

提示：**你不能通过看当前被影响的虚拟机实例的CPU性能指标来判断你所遇到的场景。（1 or 2）** 当你有很多的虚拟宿主机上分别都部署了相同职责的服务程序（可能作为集群）时，就比较容易知道自己遇到的问题了。

是否 %st(CPU Steal Time Percentage) 在所有机器上面都上涨了？

这个意味着你的虚拟机在使用更多的CPU资源。你需要为你的虚拟起增加更多的CPU资源的配额。

是否%st(CPU Steal Time Percentage) 只在一部分机器上面陡峭增长？

这个意味着物理机器被超卖了。把你自己的虚拟机挪到另一个物理机器去吧。

## 什么时候你应该担心？

一般的参考标准-**如果steal time 超过了10%并且持续了20分钟，那么虚拟机就可能性能下降了**

当这种情况发生：

关闭虚拟机并且挪到另一台物理机器上面

如果steal time维持在很高的数值， 那么增加CPU资源配额。

如果steal time维持在很高的数值， 联系你的虚拟机提供商。你的虚拟机提供商有可能在超卖物理机。

# 理解

从定义可以看出来，steal time指的是虚拟机是否获得了真实的物理cpu资源，所以steal time只在虚拟机下是有意义的，在vm内部通过top或者vmstat是可以查看到这个值。

而且，只会在cpu资源被超配的情况下才会发生。

* 对于vm使用者来说，可以判断自己的vm cpu是否是被超配了。

* 而对于iaas提供商来说，可以根据这个判断设置的超配额是否是合适的。超配的前提是：我们认为同一个hypervisor上的instance不会在同一时刻都那么忙，所以可以分着用cpu。那如果这些vm都发生了steal time比较高的情况，就说明cpu已经供应不过来了，那么应该考虑migration，并且降低超配额。

# KVM “Steal Time” support

The KVM steal time feature provides accurate data to a guest regarding CPU utilization and virtual machine performance. Large amounts of steal time indicates that the virtual machine performance is curtailed by the CPU time assigned to the guest by the hypervisor. The user can relieve the performance issues caused by CPU contention by running fewer guests on the host or by increasing the CPU priority of the guest. The KVM steal time value provides users with data to allow them to take the next step in improving their application run-time performance.

这边还有个kvm steal time的源代码实现分析：[http://oenhan.com/kvm-steal-time](http://oenhan.com/kvm-steal-time)

