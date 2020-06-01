---
layout: post
title:  "TCP队列问题"
date:   2020-06-01 23:00:00 +0800
categories: Linux
tags: Linux-Network
excerpt: TCP队列问题
mathjax: true
typora-root-url: ../
---

# TCP队列问题

发现sj环境rabbitmq, mariadb, api service都不正常

```shell
[root@ci22sjctr001 ~]# netstat -s |grep -i listen
538869 times the listen queue of a socket overflowed
550866 SYNs to LISTEN sockets dropped
```

这个socket overflowd跟dropped值一直在增长

## 三次握手

正常TCP建连接三次握手过程：

- 第一步：client 发送 syn 到server 发起握手；
- 第二步：server 收到 syn后回复syn+ack给client；
- 第三步：client 收到syn+ack后，回复server一个ack表示收到了server的syn+ack（此时client的56911端口的连接已经是established）

```shell
cat /proc/sys/net/ipv4/tcp_abort_on_overflow
0
```

![img](/../assets/images/tcp-sync-queue-and-accept-queue-small-1024x747.jpg)

这里有两个队列：

* syns queue(半连接队列）
* accept queue（全连接队列）

三次握手中，在第一步server收到client的syn后，把相关信息放到半连接队列中，同时回复syn+ack给client（第二步）；

```
比如syn floods 攻击就是针对半连接队列的，攻击方不停地建连接，但是建连接的时候只做第一步，第二步中攻击方收到server的syn+ack后故意扔掉什么也不做，导致server上这个队列满其它正常请求无法进来
```

第三步的时候server收到client的ack，如果这时全连接队列没满，那么从半连接队列拿出相关信息放入到全连接队列中，否则按tcp_abort_on_overflow指示的执行。

这时如果全连接队列满了并且tcp_abort_on_overflow是0的话，server过一段时间再次发送syn+ack给client（也就是重新走握手的第二步），如果client超时等待比较短，就很容易异常了。

tcp_abort_on_overflow

* 0  表示如果三次握手第三步的时候全连接队列满了那么server扔掉client 发过来的ack（在server端认为连接还没建立起来）
* 1 表示第三步的时候如果全连接队列满了，server发送一个reset包给client，表示废掉这个握手过程和这个连接（本来在server端这个连接就还没建立起来）

所以：

* TCP三次握手后有个accept队列，进到这个队列才能从Listen变成accept
* 满了之后握手第三步的时候server就忽略了client发过来的ack包（隔一段时间server重发握手第二步的syn+ack包给client），如果这个连接一直排不上队就异常了。

可以通过改大`net.ipv4.tcp_max_syn_backlog`和`net.core.somaxconn`来暂时解决这个问题

另外还有一个参数

`net.ipv4.tcp_syncookies = 1` 表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭