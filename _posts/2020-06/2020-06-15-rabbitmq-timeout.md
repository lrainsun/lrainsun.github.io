---
layout: post
title:  "rabbitmq cluster failed with timeout"
date:   2020-06-15 23:00:00 +0800
categories: Linux
tags: Linux-Service
excerpt: rabbitmq cluster failed with timeout
mathjax: true
typora-root-url: ../
---

# rabbitmq cluster failed with timeout

最近产线出现了rabbitmq cluster故障，整个cluster没办法正常工作，rabbitmq有超时以及其他一些错误打印

```shell
2020-06-15 02:34:14.730 [error] <0.19764.8> closing AMQP connection <0.19764.8> (10.252.126.79:33602 -> 10.252.126.79:5672 - neutron-server:250535:bb1f6afb-0c3c-4442-a74c-64d7129a6896):
missed heartbeats from client, timeout: 60s
```

这个heartbeat是rabbitmq server跟client之间的，在server端跟client端都有配置，互相协商，我们本想过改大server端配置，可是查了一下没用，因为如果client端还是60s的配置，那么经过协商之后还是会维持短的那个60s。而为啥会出现timeout呢，还怀疑是不是client端处理太慢了？可是看起来好像又是server端的问题

```shell
operation queue.declare caused a channel exception not_found: failed to perform operation on queue 'neutron-vo-SubPort-1.0_fanout_6985da1c44164a8b9c926397af8abfb9' in vhost '/' due to timeout
```

这个timeout Google了下说是已经问题，3.6.6已经fix了，可是我们用的版本已经是3.7.21了，又看到说需要跟erlang版本相匹配，查了3.7.21的匹配版本是21.3跟22.X，我们用的是22.1.6，也匹配

![image-20200615203544585](/../assets/images/image-20200615203544585.png)

```shell
2020-06-15 02:28:05.046 [warning] <0.28609.16> rabbit_sysmon_handler busy_dist_port <0.28671.16> [{name,rabbit},{initial_call,{application_master,start_it,4}},{erlang,bif_return_trap,2},{message_queue_len,1}] {#Port<0.63457>,unknown}
```

还有很多busy_dist_port打印，这个通过在`/etc/rabbitmq/rabbitmq-env.conf`增加配置

```
ABBITMQ_SERVER_ADDITIONAL_ERL_ARGS="+zdbbl 192000"
```

可以有效改善，但看起来对cluster的恢复并无帮助

最初是controller1上的rabbitmq process down了，down的前后看似有比较高的cpu load，2跟3看起来还是可以正常工作，因为是primary的，我们想要把它加回去，还需要follow这样一个流程

```shell
On ci22sjctr002
rabbitmqctl forget_cluster_node rabbit@ci22sjctr001

On ci22sjctr001
rm -rf /var/lib/rabbitmq/mnesia/*
service rabbitmq-server start
rabbitmqctl stop_app
rabbitmqctl join_cluster rabbit@ci22sjctr002
rabbitmqctl start_app
```

但是加进去之后，很奇怪的是，原本正常的cluster反正不能正常工作了，不得已又把1上的rabbitmq停掉了。

而2跟3运行了一段时间之后，也又出现了同样的问题，尝试了很多动作都没办法把cluster修复好，我们推测是跟node之前做mirror有关，所以决定只剩一台试一下，最后发现确实如预期的，一台rabbitmq由于并不需要做mirror的动作，反而可以正常工作了。

我们配置的mirror policy是all

![image-20200615204614924](/../assets/images/image-20200615204614924.png)

也就是三台rabbitmq之前都会互相做mirror，对于每个queue都有一个master node，而它需要负责向另外两个slave node做mirror

![image-20200615204742519](/../assets/images/image-20200615204742519.png)

推测应该是我们环境规模比较大，数量也比较大，所以这个mirror的动作变成了压死rabbitmq的稻草。然后发现有个配置

![image-20200615204841448](/../assets/images/image-20200615204841448.png)

做mirror的batch size，我们觉得改大可能有效果。在dev-ocp2想尝试一下，但dev-ocp2本就没出问题，于是把dev-ocp2改小，发现成功复现了产线问题，如上所述的错误日志都出现了，那基本可以肯定是这个问题。另外还有个net_ticktime时间，感觉上改大也是可以有改善的，但我们从60改到120，并没解决问题，不知道是没效果还是应该继续再改大。

另外collect_statistics_interval也可以配置大一些，我们不需要那么频繁地去获取statistic

而rabbitmq的配置，在rabbitmq 3.7.0 之前，rabbitmq.config使用了Erlang语法配置格式，新的版本使用了sysctl 格式rabbitmq.conf

所以如果是erlang语法，就写到rabbitmq.config里，比如

```erlang
[
  {rabbit, [
    {cluster_partition_handling, pause_minority},
    {tcp_listen_options, [binary, {packet, raw}, {reuseaddr, true}, {backlog, 128}, {nodelay, true}, {exit_on_close, false}, {keepalive, true}]},
    {mirroring_sync_batch_size, 8192},
    {collect_statistics_interval, 30000}
  ]},
  {kernel, [
    {inet_dist_listen_max, 35672},
    {inet_dist_listen_min, 35672}
  ]}
,
  {rabbitmq_management, [
    {listener, [
      {port, 15672}
    ]}
  ]}
].
```

如果是sysctl无法，就写到rabbitmq.conf里，比如

```shell
cluster_partition_handling = pause_minority

tcp_listen_options.backlog = 128
tcp_listen_options.nodelay = true
tcp_listen_options.exit_on_close = false
tcp_listen_options.keepalive = true

collect_statistics_interval = 30000

management.tcp.port = 15672

mirroring_sync_batch_size = 2048
```

除此之外，还有一些问题以及配置需要详细研究。

# References

[1] [https://www.rabbitmq.com/ha.html](https://www.rabbitmq.com/ha.html)

[2] [https://www.rabbitmq.com/configure.html](https://www.rabbitmq.com/configure.html)

[3] [https://www.rabbitmq.com/ha.html#batch-sync]( https://www.rabbitmq.com/ha.html#batch-sync)

[4] [https://www.rabbitmq.com/nettick.html](https://www.rabbitmq.com/nettick.html)