---
layout: post
title:  "Linux Module"
date:   2020-03-28 23:00:00 +0800
categories: Openstack
tags: Linux-Kernel
excerpt: Linux Module
mathjax: true
typora-root-url: ../
---

# Linux Module

linux 2.0版本以后都支持模块化，因为内核中的一部分常驻在内存中，（如最常使用的进程，如scheduler等），但是其它的进程只是在需要的时候才被载入。这种按需加载，就是模块。它能使内核保持较小的体积，模块另一个优点是内核可以动态的加载、卸载它们（或者自动地由kerneld守护程序完成）。

查看已经加载的linux kernel module

`lsmod` or `cat /proc/modules`

`dmesg`可以查看linux kernel module中的打印

## 内核模块代码

Linux内核模块都是用C语言编写的，而内核模块跟普通应用程序有一些区别。

* 内核模块没有main()函数

* 非顺序执行：

  ```
  内核模块使用初始化函数将自身注册并处理请求，初始化函数运行后就结束了。
  内核模块处理的请求在模块代码中定义。这和常用于图形用户界面（graphical-user interface，GUI）应用的事件驱动编程模型比较类似
  ```

* 没有自动清理：

  ```
  任何由内核模块申请的内存，必须要模块卸载时手动释放，否则这些内存将无法使用，直到系统重启。 
  ```

* 不要使用 printf() 函数：

  ```
  内核代码无法访问为 Linux 用户空间编写的库。内核模块运行在内核空间，它有自己独立的地址空间。内核空间和用户空间的接口被清晰的定义和控制。
  内核模块可以通过 printk() 函数输出信息，这些输出可以在用户空间查看到。 
  ```

* 会被中断：

  ```
  内核模块一个概念上困难的地方在于他们可能会同时被多个程序 / 进程使用。构建内核模块时需要小心，以确保在发生中断的时候行为一致和正确。
  ```

* 更高级的执行特权：

  ```
  通常内核模块会比用户空间程序分配更多的 CPU 周期。这看上去是一个优势，然而需要特别注意内核模块不会影响到系统的综合性能。
  ```

* 无浮点支持：

  ```
  对用户空间应用，内核代码使用陷阱（trap）来实现整数到浮点模式的转换。然而在内核空间中这些陷阱难以使用。
  替代方案是手工保存和恢复浮点运算，这是最好的避免方式，并将处理留给用户空间代码。
  ```

# 编译内核模块

构建内核模块需要 Makefile 文件，事实上是一个特殊的 `kbuild Makefile` 。

openvswitch的Makefile文件

```makefile
#
# Makefile for Open vSwitch.
#

obj-$(CONFIG_OPENVSWITCH) += openvswitch.o

openvswitch-y := \
	actions.o \
	datapath.o \
	dp_notify.o \
	flow.o \
	flow_netlink.o \
	flow_table.o \
	vport.o \
	vport-internal_dev.o \
	vport-netdev.o

ifneq ($(CONFIG_NF_CONNTRACK),)
openvswitch-y += conntrack.o
endif

obj-$(CONFIG_OPENVSWITCH_VXLAN)+= vport-vxlan.o
obj-$(CONFIG_OPENVSWITCH_GENEVE)+= vport-geneve.o
obj-$(CONFIG_OPENVSWITCH_GRE)	+= vport-gre.o
```

* `obj-$(CONFIG_OPENVSWITCH) += openvswitch.o`是目标定义，定义了需要构建的模块（openvswitch.o）
* `openvswitch-y`表示内置的对象目标

Makefile 文件中需要提醒的内容和普通 Makefile 文件类似。 `$(shell uname -r) 命令返回当前内核构建版本`，这确保了一定程度的可移植性。 `-C` 选项在执行任何 make 任务前将目录切换到内核目录，它在那里会发现内核的顶层 makefile。 `M=$(PWD)` 变量赋值告诉 make 命令实际工程文件存放位置，在试图建立模块 (modules) 目标前，回到你的模块源码目录，而此目标会在 `obj-m` 变量里面找模块列表。对于外部内核模块来说，modules 目标是默认目标。另一种目标是 modules_install，它将安装模块（make命令必须使用超级用户权限执行且需要提供模块安装路径）。

## 宏 EXPORT_SYMBOL

Linux-2.4之前，默认的非static 函数和变量都会自动导入到kernel 空间， 而Linux-2.6之后默认不导出所有的符号，所以使用 `EXPORT_SYMBOL()` 做标记。

`EXPORT_SYMBOL` 标签内定义的函数或者符号对全部内核代码公开，不用修改内核代码就可以在内核模块中直接调用。
即使用 `EXPORT_SYMBOL` 可以将一个函数以符号的方式导出给其他模块使用。
符号的意思就是函数的入口地址，或者说是把这些符号和对应的地址保存起来的，在内核运行的过程中，可以找到这些符号对应的地址的。

## EXPORT_SYMBOL使用方法

```
1.在模块函数定义之后使用 `EXPORT_SYMBOL(函数名)`
2.在调用该函数的模块中使用 `extern` 对要使用的符号或者函数进行声明
3.首先加载定义该函数的模块，再加载调用该函数的模块
```

## EXPORT_SYMBOL示范

比如有两个驱动模块：Module A和Module B，其中Module B使用了Module A中的export的函数，因此在Module B的Makefile文件中必须添加：

```
KBUILD_EXTRA_SYMBOLS += /path/to/ModuleA/Module.symvers
export KBUILD_EXTRA_SYMBOLS
```

# 编译Openvswitch linux kernel module

首先需要下载并安装对应版本的kernel-devel包（kernel-devel包只包含用于内核开发环境所需的内核头文件以及Makefile），比如我们要编译的版本是`3.10.0-1062.12.1`

```shell
[root@ci81hf1cmp008 ovs]# wget http://mirror.centos.org/centos/7/updates/x86_64/Packages/kernel-devel-3.10.0-1062.9.1.el7.x86_64.rpm

[root@ci81hf1cmp008 ovs]# rpm -ivh kernel-devel-3.10.0-1062.9.1.el7.x86_64.rpm --force
warning: kernel-devel-3.10.0-1062.9.1.el7.x86_64.rpm: Header V3 RSA/SHA256 Signature, key ID f4a80eb5: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:kernel-devel-3.10.0-1062.9.1.el7 ################################# [100%]
```

解压后的代码会被放在这里

```shell
[root@ci81hf1cmp008 openvswitch]# ls -al /usr/src/kernels/3.10.0-1062.9.1.el7.x86_64/
total 4880
drwxr-xr-x   22 root root    4096 Mar 29 03:17 .
drwxr-xr-x.   4 root root      75 Mar 29 03:17 ..
drwxr-xr-x   32 root root    4096 Mar 29 03:17 arch
drwxr-xr-x    3 root root      78 Mar 29 03:17 block
-rw-r--r--    1 root root  153186 Dec  6 15:53 .config
drwxr-xr-x    4 root root      76 Mar 29 03:17 crypto
drwxr-xr-x  119 root root    4096 Mar 29 03:17 drivers
drwxr-xr-x    2 root root      22 Mar 29 03:17 firmware
drwxr-xr-x   75 root root    4096 Mar 29 03:17 fs
drwxr-xr-x   28 root root    4096 Mar 29 03:17 include
drwxr-xr-x    2 root root      37 Mar 29 03:18 init
drwxr-xr-x    2 root root      22 Mar 29 03:18 ipc
-rw-r--r--    2 root root     505 Feb  4 23:07 Kconfig
drwxr-xr-x   13 root root     247 Mar 29 03:18 kernel
drwxr-xr-x   10 root root     219 Mar 29 03:18 lib
-rw-r--r--    1 root root   51297 Dec  6 15:53 Makefile
-rw-r--r--    2 root root    2305 Feb  4 23:07 Makefile.qlock
drwxr-xr-x    2 root root      58 Mar 29 03:18 mm
-rw-r--r--    1 root root 1141069 Dec  6 15:53 Module.symvers
drwxr-xr-x   61 root root    4096 Mar 29 03:18 net
drwxr-xr-x   15 root root     231 Mar 29 03:18 samples
drwxr-xr-x   13 root root    4096 Mar 29 03:18 scripts
drwxr-xr-x    9 root root     136 Mar 29 03:18 security
drwxr-xr-x   24 root root     301 Mar 29 03:18 sound
-rw-r--r--    1 root root 3599385 Dec  6 15:53 System.map
drwxr-xr-x   20 root root     254 Mar 29 03:18 tools
drwxr-xr-x    2 root root      37 Mar 29 03:18 usr
drwxr-xr-x    4 root root      44 Mar 29 03:18 virt
-rw-r--r--    1 root root      41 Dec  6 15:53 vmlinux.id
```

openvswitch相关在这里

```shell
[root@ci81hf1cmp008 ovs]# ls -al /usr/src/kernels/3.10.0-1062.9.1.el7.x86_64/net/openvswitch/
total 12
drwxr-xr-x  2 root root   37 Mar 28 07:01 .
drwxr-xr-x 61 root root 4096 Mar 28 07:01 ..
-rw-r--r--  1 root root 2295 Feb  4 23:07 Kconfig
-rw-r--r--  1 root root  446 Feb  4 23:07 Makefile
```

我们还需要下载对应kernel的源码包并解压，并解压

```shell
wget https://buildlogs.centos.org/c7.1908.u.x86_64/kernel/20191206154625/3.10.0-1062.9.1.el7.x86_64/kernel-3.10.0-1062.9.1.el7.src.rpm
rpm2cpio ../kernel-3.10.0-1062.9.1.el7.src.rpm  | cpio -iv
```

openvswitch datapath代码在

```shell
[root@ci81hf1cmp008 linux-3.10.0-1062.9.1.el7]# ls -al net/openvswitch/
total 388
drwxrwxr-x  2 root root  4096 Dec  2 13:08 .
drwxrwxr-x 61 root root  4096 Dec  2 13:08 ..
-rw-rw-r--  1 root root 34797 Dec  2 13:08 actions.c
-rw-rw-r--  1 root root 46684 Dec  2 13:08 conntrack.c
-rw-rw-r--  1 root root  3076 Dec  2 13:08 conntrack.h
-rw-rw-r--  1 root root 61689 Dec  2 13:08 datapath.c
-rw-rw-r--  1 root root  6746 Dec  2 13:08 datapath.h
-rw-rw-r--  1 root root  2664 Dec  2 13:08 dp_notify.c
-rw-rw-r--  1 root root 22699 Dec  2 13:08 flow.c
-rw-rw-r--  1 root root  8400 Dec  2 13:08 flow.h
-rw-rw-r--  1 root root 78085 Dec  2 13:08 flow_netlink.c
-rw-rw-r--  1 root root  2933 Dec  2 13:08 flow_netlink.h
-rw-rw-r--  1 root root 19310 Dec  2 13:08 flow_table.c
-rw-rw-r--  1 root root  2883 Dec  2 13:08 flow_table.h
-rw-rw-r--  1 root root  2295 Dec  2 13:08 Kconfig
-rw-rw-r--  1 root root   446 Dec  2 13:08 Makefile
-rw-rw-r--  1 root root 13242 Dec  2 13:08 vport.c
-rw-rw-r--  1 root root  3187 Dec  2 13:08 vport-geneve.c
-rw-rw-r--  1 root root  2799 Dec  2 13:08 vport-gre.c
-rw-rw-r--  1 root root  6312 Dec  2 13:08 vport.h
-rw-rw-r--  1 root root  7644 Dec  2 13:08 vport-internal_dev.c
-rw-rw-r--  1 root root  1058 Dec  2 13:08 vport-internal_dev.h
-rw-rw-r--  1 root root  5482 Dec  2 13:08 vport-netdev.c
-rw-rw-r--  1 root root  1155 Dec  2 13:08 vport-netdev.h
-rw-rw-r--  1 root root  4343 Dec  2 13:08 vport-vxlan.c
```

把datapath源码拷贝到一个目录作为编译目录，把Makefile文件拷过来（Makefile可能源码包里就有，也可以不拷贝），此外，为了解决load module时的问题

```shell
[62972.099953] openvswitch: no symbol version for module_layout
```

还需要将如下文件（就在kernel-devel包里，拷贝到我们编译目录下）

```shell
[root@ci81hf1cmp008 openvswitch]# cp /usr/src/kernels/3.10.0-1062.9.1.el7.x86_64/net/openvswitch/* .
[root@ci81hf1cmp008 openvswitch]# cp /usr/src/kernels/3.10.0-1062.9.1.el7.x86_64/Module.symvers .
```

然后可以进到`/usr/src/kernerls/3.10.0-1062.9.1.el7.x86_64`开始编译

```shell
[root@ci81hf1cmp008 3.10.0-1062.9.1.el7.x86_64]# make M=/root/ovs/openvswitch
  LD      /root/ovs/openvswitch/built-in.o
  CC [M]  /root/ovs/openvswitch/actions.o
  CC [M]  /root/ovs/openvswitch/datapath.o
  CC [M]  /root/ovs/openvswitch/dp_notify.o
  CC [M]  /root/ovs/openvswitch/flow.o
  CC [M]  /root/ovs/openvswitch/flow_netlink.o
  CC [M]  /root/ovs/openvswitch/flow_table.o
  CC [M]  /root/ovs/openvswitch/vport.o
  CC [M]  /root/ovs/openvswitch/vport-internal_dev.o
  CC [M]  /root/ovs/openvswitch/vport-netdev.o
  CC [M]  /root/ovs/openvswitch/conntrack.o
  LD [M]  /root/ovs/openvswitch/openvswitch.o
  CC [M]  /root/ovs/openvswitch/vport-vxlan.o
  CC [M]  /root/ovs/openvswitch/vport-geneve.o
  CC [M]  /root/ovs/openvswitch/vport-gre.o
  Building modules, stage 2.
  MODPOST 4 modules
  CC      /root/ovs/openvswitch/openvswitch.mod.o
  LD [M]  /root/ovs/openvswitch/openvswitch.ko
  CC      /root/ovs/openvswitch/vport-geneve.mod.o
  LD [M]  /root/ovs/openvswitch/vport-geneve.ko
  CC      /root/ovs/openvswitch/vport-gre.mod.o
  LD [M]  /root/ovs/openvswitch/vport-gre.ko
  CC      /root/ovs/openvswitch/vport-vxlan.mod.o
  LD [M]  /root/ovs/openvswitch/vport-vxlan.ko
```

或者呢，也可以不进到目录，这样：

```shell
[root@ci81hf1cmp008 ~]# make -C /usr/src/kernels/3.10.0-1062.9.1.el7.x86_64/ modules M=/root/ovs/openvswitch
```

而看起来，`make`或者`make modules`都可以

> *Unlike kernels before 2.6, you no longer need to run make dep before building the kernel—the dependency tree is maintained automatically.You also do not need to specify a specific build type, such as bzImage, or build modules separately, as you did in old versions.The default Makefile rule will handle everything.*
>
> 也就是在`2.6`版本以后的`Linux kernel`中，执行`make`或`make all`命令即包含了`make modules`。

或者 

```shell
[root@ci81hf1cmp008 ~]# make -C /lib/modules/`uname -r`/build M=/root/openvswitch

[root@ci81hf1cmp008 ~]# ls -al /lib/modules/`uname -r`/build
lrwxrwxrwx 1 root root 43 Feb  3 09:21 /lib/modules/3.10.0-1062.9.1.el7.x86_64/build -> /usr/src/kernels/3.10.0-1062.9.1.el7.x86_64
```

# Reference

[1] [https://juniorprincewang.github.io/2018/11/16/%E7%BC%96%E5%86%99Linux%E5%86%85%E6%A0%B8%E6%A8%A1%E5%9D%97/](https://juniorprincewang.github.io/2018/11/16/编写Linux内核模块/)

