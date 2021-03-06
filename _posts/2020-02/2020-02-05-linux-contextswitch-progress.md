---
layout: post
title:  "CPU上下文切换之进程上下文切换"
date:   2020-02-05 21:00:00 +0800
categories: Linux
tags: Linux-Performance
excerpt: CPU上下文切换之进程上下文切换
mathjax: true
typora-root-url: ../
---

# 内核空间和用户空间

Linux 按照特权等级，把进程的运行空间分为内核空间和用户空间

![image-20200205214636090](/../assets/images/image-20200205214636090.png)

* 内核空间（Ring 0）：具有最高权限，可以直接访问所有资源；
* 用户空间（Ring 3）：只能访问受限资源，不能直接访问内存等硬件设备，必须通过系统调用陷入到内核中，才能访问这些特权资源。

进程既可以在用户空间运行，又可以在内核空间中运行。进程在用户空间运行时，被称为进程的用户态，而陷入内核空间的时候，被称为进程的内核态。

# 系统调用

从用户态到内核态的转变，需要通过系统调用来完成。

系统调用的过程也会发生 CPU 上下文的切换：

* CPU 寄存器里原来用户态的指令位置，需要先保存起来。
* 为了执行内核态代码，CPU 寄存器需要更新为内核态指令的新位置。
* 跳转到内核态运行内核任务。

而系统调用结束后，CPU 寄存器需要恢复原来保存的用户态，然后再切换到用户空间，继续运行进程。

***一次系统调用的过程，其实是发生了两次 CPU 上下文切换***

系统调用（特权模式切换）不是进程上下文切换：

* 不涉及虚拟内存等进程用户态资源
* 一直是同一个进程在运行

# 进程上下文切换

进程是由内核来管理和调度的，进程的切换只能发生在内核态。

进程的上下文包括：

* 虚拟内存、栈、全局变量等用户空间的资源
* 内核堆栈、寄存器等内核空间的状态。

![image-20200205220353023](/../assets/images/image-20200205220353023.png)

进程的上下文切换比系统调用时多了一步：在保存当前进程的内核状态和 CPU 寄存器之前，需要先把该进程的虚拟内存、栈等保存下来；而加载了下一进程的内核态后，还需要刷新进程的虚拟内存和用户栈。

保存上下文和恢复上下文的过程并不是“免费”的，需要内核在 CPU 上运行才能完成。如果进程上下文切换次数过多，就容易导致负载升高。

# 进程上下文切换时机

进程切换时才需要切换上下文，换句话说，只有在进程调度的时候，才需要切换上下文。

Linux 为每个 CPU 都维护了一个就绪队列，将活跃进程（即正在运行和正在等待CPU 的进程）按照优先级和等待 CPU 的时间排序，然后选择最需要 CPU 的进程，也就是优先级最高和等待 CPU 时间最长的进程来运行。

* 为了保证所有进程可以得到公平调度，CPU 时间被划分为一段段的时间片，这些时间片再被轮流分配给各个进程。这样，当某个进程的时间片耗尽了，就会被系统挂起，切换到其它正在等待 CPU 的进程运行。

* 进程在系统资源不足（比如内存不足）时，要等到资源满足后才可以运行，这个时候进程也会被挂起，并由系统调度其他进程运行。

* 当进程通过睡眠函数 sleep 这样的方法将自己主动挂起时，也会重新调度。

* 当有优先级更高的进程运行时，为了保证高优先级进程的运行，当前进程会被挂起，由高优先级进程来运行。

* 发生硬件中断时，CPU 上的进程会被中断挂起，转而执行内核中的中断服务程序。