---
layout: post
title:  "Prometheus AlertManager HA"
date:   2020-02-03 23:00:00 +0800
categories: Prometheus
tags: Prometheus-AlertManager
excerpt: Prometheus AlertManager HA
mathjax: true
---

# AlertManager HA

为了提高prometheus的高可用性，通常我们会用多个prometheus server，而不同的prometheus server会发送相同的告警给alertmanager，经过去重，分组，路由，alertmanager可以处理从不同prometheus server发过来的告警。

![img](../assets/images/1544_nwc1dt_XRWKERT.jpg!r800x0.jpg)

但是alertmanager本身也会产生单点故障，所以我们还需要配置多个alertmanager，每个prometheus server都会往所有的alertmanager发送告警。而alertmanager彼此不知道对方的存在，就会出现告警通知被不同的alertmanager重复发送的问题。

![img](../assets/images/1544_s9uGXH_XRWKERT.jpg!r800x0.jpg)

# Gossip协议

为了解决上面的问题，采用了Gossip协议，Gossip协议用于实现分布式节点之间的信息交换和状态同步。

* push-based：当集群中某一节点A完成一个工作后，随机的从其它节点B并向其发送相应的消息，节点B接收到消息后在重复完成相同的工作，直到传播到集群中的所有节点
* pull-based：节点A会随机的向节点B发起询问是否有新的状态需要同步，如果有则返回

# AlertManager的应用

![img](../assets/images/22C2715D5DAECF6C8DCF8C7EB8E0B519.png)

## 通知流水线

* 在第一个阶段Silence中，Alertmanager会判断当前通知是否匹配到任何的静默规则，如果没有则进入下一个阶段，否则则中断流水线不发送通知。
* 在第二个阶段Wait中，Alertmanager会根据当前Alertmanager在集群中所在的顺序(index)等待index * 5s的时间。
* 当前Alertmanager等待阶段结束后，Dedup阶段则会判断当前Alertmanager数据库中该通知是否已经发送，如果已经发送则中断流水线，不发送告警，否则则进入下一阶段Send对外发送告警通知。
* 告警发送完成后该Alertmanager进入最后一个阶段Gossip，Gossip会通知其他Alertmanager实例当前告警已经发送。其他实例接收到Gossip消息后，则会在自己的数据库中保存该通知已发送的记录。

## alertmanager里的Gossip

![img](../assets/images/78FDB77787D867798F911D7BBF73D600.png)

Silence设置同步：Alertmanager启动阶段基于Pull-based从集群其它节点同步Silence状态，当有新的Silence产生时使用Push-based方式在集群中传播Gossip信息。

通知发送状态同步：告警通知发送完成后，基于Push-based同步告警发送状态。Wait阶段可以确保集群状态一致。

# 问题

在我们的场景里，有三台alertmanager做HA，发生了告警被重复发送的问题。

这边有三个配置：

```shell
--cluster.peer-timeout=15s
Time to wait between peers to send notifications.
--cluster.gossip-interval=200ms
Interval between sending gossip messages. By lowering this value (more frequent) gossip messages are propagated across the cluster more quickly at the expense of increased bandwidth.
--cluster.pushpull-interval=1m0s
Interval for gossip state syncs. Setting this interval lower (more frequent) will increase convergence speeds across larger clusters at the expense of increased bandwidth usage.
```

 看一下第一个配置，我们的告警会被发送到email, webhook, 这个时间有时候可能会比较久一些。那么其他alertmanager会等待peer-timeout（默认15s）的时间，如果在这个时间内还没发送成功，Gossip的消息还没有传送过来，就认为该alert还没有被发送，就会尝试去再次发送。

需要适当改大timeout配置，可以解决这个问题

