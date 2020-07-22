---
layout: post
title:  "跨DC ETCD"
date:   2020-07-22 23:00:00 +0800
categories: Kubernetes
tags: Kubernetes-Etcd
excerpt: 跨DC ETCD
mathjax: true
typora-root-url: ../
---

# 跨DC ETCD

跨dc的etcd部署可以提高容错性，代价是会有比较高的请求延迟，由于etcd依赖于成员仲裁来达成共识，因此跨数据中心的延迟将在某种程度上变得显著，因为至少大多数集群成员必须响应共识请求。此外，群集数据必须在所有member之间复制，因此也会产生带宽成本。

在local network etcd的默认配置可以很好的工作，但是如果是跨dc的，heartbeat interval和election timeout需要tunning

network并不是唯一的延迟，每个request & response还可能受leader & follower的slow disk影响

所有这些timeout代表了总的时间from request to successful response from the other machine

# 影响参数

底层的分布式一致协议依赖于两个参数：

* heartbeat interval：This is the frequency with which the leader will notify followers that it is still the leader. leader每隔heartbeat时间会通知followers，自己还是leader
  最佳实践是，member间往返的时间设置为heartbeat interval，默认是100ms
* election timeout： This timeout is how long a follower node will go without hearing a heartbeat before attempting to become leader itself. 当follower没有收到header的heartbeat消息的时候，过多久会尝试自己担当leader，默认是1000ms

# 参数设置

这两个参数的设置需要一个平衡，如果设置的太小，etcd就会发送很多无用的消息，增加cpu和network的浪费；而如果设置的太大，就会需要更长的时间才能发现一个leader的failure。通常需要根据member之间的round-trip time(RTT)来设置。

* heartbeat interval通常设置成member之间最大RTT的0.5-1.5倍
* election timeout根据heartbeat interval和平均RTT，至少是10倍的RTT，可以解决网络的差异

# 参数上限

在全球分布的etcd的集群里，在美国大陆的合理的RTT值在130ms，美国跟日本之间RTT大概在350-400ms，如果网络性能不均匀或有规则的数据包延迟/丢失，则可能需要重试几次才能成功发送数据包，所以5s可能是比较安全的global RTT的上限值

而对于election timetout来说会大一个数量级，也就是50s是一个比较合理的上限值

heartbeat interval and election timeout在一个cluster里应该是一致的，否则会打破cluster的稳定性

# References

[1] [https://etcd.io/docs/v3.4.0/tuning/](https://etcd.io/docs/v3.4.0/tuning/)

