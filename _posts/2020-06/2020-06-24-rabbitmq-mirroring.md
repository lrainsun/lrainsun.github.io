---
layout: post
title:  "rabbitmq queue mirroring"
date:   2020-06-24 23:00:00 +0800
categories: Linux
tags: Linux-Service
excerpt: rabbitmq queue mirroring
mathjax: true
typora-root-url: ../
---

# rabbitmq queue mirroring

默认，rabbitmq cluster的一个queue的内容只在一个node上（queue在哪里被声明），这跟exchanges和bindings不一样（在所有节点）。我们可以配置queue在多个节点之间做mirror。

每个mirroed queue都有一个master和一个或多个mirrors。所有关于queue的操作都会被apply到queue的master节点并传播给mirrors，包括enqueueing publishes, delivering messages to consumers, tracking acks from consumers等等。

发布到queueq的messages会被复制到所有的mirrors，consumers不管连接到哪个node，都会连接master queue。所以mirroring只是增强了ha，并没有load balance（所以mirror的节点都需要做所有的工作）

如果queue的master node挂了，那么最先被mirror的节点会被promote成新的master，

# 配置mirroring

我们可以通过policies来配置mirroring

![image-20200624105655344](/../assets/images/image-20200624105655344.png)

Key policy attributes are

- name: it can be anything but ASCII-based names without spaces are recommended
- pattern: a regular expression that matches one or more queue (exchange) names. Any regular expression can be used.
- definition: a set of key/value pairs (think a JSON document) that will be injected into the map of optional arguments of the matching queues and exchanges
- priority: see below

通过policy可以enable mirroring，任何时间都可以改变policy，从non-mirrored到mirror，或者反过来。但是non-mirrored queue跟一个没有mirror的mirrored queue是有差别的，non-mirrored queue因为没有额外的mirroring infrastructure，所以会提供更高的throughput。

| ha-mode | ha-params    | Result                                                       |
| :------ | :----------- | :----------------------------------------------------------- |
| exactly | *count*      | Number of queue replicas (master plus mirrors) in the cluster. A *count* value of 1 means a single replica: just the queue master. If the node running the queue master becomes unavailable, [the behaviour depends on queue durability](https://www.rabbitmq.com/ha.html#non-mirrored-queue-behavior-on-node-failure). A *count* value of 2 means 2 replicas: 1 queue master and 1 queue mirror. In other words: `NumberOfQueueMirrors = NumberOfNodes - 1`. If the node running the queue master becomes unavailable, the queue mirror will be automatically promoted to master according to the [mirror promotion strategy](https://www.rabbitmq.com/ha.html#unsynchronised-mirrors) configured. If there are fewer than *count* nodes in the cluster, the queue is mirrored to all nodes. If there are more than *count* nodes in the cluster, and a node containing a mirror goes down, then a new mirror will be created on another node. Use of `exactly` mode with [`"ha-promote-on-shutdown": "always"`](https://www.rabbitmq.com/ha.html#cluster-shutdown) can be dangerous since queues can migrate across a cluster and become unsynced as it is brought down. |
| all     | (none)       | Queue is mirrored across all nodes in the cluster. When a new node is added to the cluster, the queue will be mirrored to that node. This setting is very conservative. Mirroring to a quorum (N/2 + 1) of cluster nodes is [recommended instead](https://www.rabbitmq.com/ha.html#replication-factor). Mirroring to all nodes will put additional strain on all cluster nodes, including network I/O, disk I/O and disk space usage. |
| nodes   | *node names* | Queue is mirrored to the nodes listed in *node names*. Node names are the Erlang node names as they appear in rabbitmqctl cluster_status; they usually have the form "`rabbit@hostname`". If any of those node names are not a part of the cluster, this does not constitute an error. If none of the nodes in the list are online at the time when the queue is declared then the queue will be created on the node that the declaring client is connected to. |

Whenever the HA policy for a queue changes it will endeavour to keep its existing mirrors as far as this fits with the new policy. 这个意思是会保持原有的mirros吗？还没太明白

一般不需要全部mirror，这样会造成network i/o, disk i/o的压力，推荐 nodes in a 3 node cluster or 3 nodes in a 5 node cluster.

# list queues

```shell
[root@ci81hf1cmp001 ~]# rabbitmqctl list_queues name policy pid slave_pids
Timeout: 60.0 seconds ...
Listing queues for vhost / ...
name	policy	pid	slave_pids
q-agent-notifier-port-update.ci81hf1cmp009.qa.webex.com	ha-all	<rabbit@ci81hf1cmp001.1.10309.2>	[<rabbit@ci81hf1cmp003.2.7061.2>, <rabbit@ci81hf1cmp002.2.7425.2>]
q-agent-notifier-security_group-update.ci81hf1cmp007.qa.webex.com	ha-all	<rabbit@ci81hf1cmp001.1.9722.2>	[<rabbit@ci81hf1cmp003.2.6524.2>, <rabbit@ci81hf1cmp002.2.6900.2>]
```

# queue master

在rabbitmq里的每个queue都有一个primary的replica，称之为queue master。所有关于这个queue的操作都会先经过master，然后再复制到其他mirrors，这样就保证了消息的有序性。

至于queue master是放在哪个node上，可以通过`x-queue-master-locator`来定义，在policy中设置`queue-master-locator`或者在配置文件里定义：

* min-master：选择绑定master最少的node
* client-local：选择定义这个queue的client连接的node
* radom：随机选择

设置'node' policy会使得当前存在的master消息（如果跟新policy不匹配的话），为了避免消息丢失，rabbitmq会保留当前的master直到一个新的mirror被同步。

> if a queue is on [A B] (with A the master), and you give it a nodes policy telling it to be on [C D], it will initially end up on [A C D]. As soon as the queue synchronises on its new mirrors [C D], the master on A will shut down.

# exclusive queues的mirroring

exclusive queue在connection被关闭之后会被删除，所以没有必要做mirror，一旦这个queue的host down了，连接总是会关闭，那么queue也就相应会被deleted。

所以exclusive queue不需要被mirrored，也不需要是持久化的

# non-mirrored queue

如果一个queue的master node是available的，那么所有的queue operatirons（比如定义，绑定，消费者管理，路由管理）等是可以在任何node上进行的，cluster node会把这些operations路由到master node，这对client来说是透明的

如果一个queue的master node是unavailable的，一个non-mirrored的queue的行为根据他的durability来决定。一个durable queue在node回来之前都是unavailable的，所有operations在这个时候都会返回类似以下的错误：

```shell
operation queue.declare caused a channel exception not_found: home node 'rabbit@hostname' of durable queue 'queue-name' in vhost '/' is down or inaccessible
```

一个non-durable的queue会被删除。

# mirrored queue的实现

所有的mirroed queue都有一个master replica和几个mirrors，mirrors跟master一样会按照一样的顺序apply operations，而且把queue维护在相同的状态。除了publishes的所有action都会到master，然后master会广播actions到mirrors。所以client从mirrored queue消费其实是从master在消费。

如果一个mirror fail，除了记录之外通常做不了什么事情，master还是master，client不需要做什么动作也不会被通知mirror的failure。mirror的failures不会马上被发现，per-connection flow control mechanism的中断可能会延迟消息的发布。

如果master fails，某一个mirror就会被promoted到master

* 运行时间最长的mirror（通常运行时间最长，那么也会跟master同步的最好，这样的假设），如果没有mirror跟master synchronised，只在master存在的message就会被丢失
* mirror认为所有之前的consumers都被突然的disconnected了，它会去requeue那些已经发送给client但是状态是pending acknowledgement的消息（可能也包括：client已经发送ack，但是由于网络问题还没到master就丢失了；或者当master广播到mirror的时候被丢失了），不管是哪种情况，新的master都别无选择，只能requeue所有没有ack的message
* 如果consumers要求在queue failed的时候被通知到，那么会通知它
* 作为requeuing的结果，client从queue re-consume要意识到，他们可能会收到之前已经收到的消息
* 当被选择的mirror变成master之后，在这期间任何被发布到mirrored queue的消息都不会被丢失（除非promote成master之后发生后续故障）。message发布到host queue mirror会被route到queue master然后被复制到所有的mirrors。如果master fail了，发到mirros的message当master promotion完成后会被加到queue
* 使用publisher confirm的client，即使master fail了，也仍然会被confirm。对于发布者来说，发布到mirrored queue与发布到non-mirrored queue没有什么不同。

如果consumers使用了automatic acknowledgement mode，message就可能被丢失。对于non-mirrored queue来说没有区别，一旦消息以自动确认方式发送给消费者，就会认为该消息已确认。

# publisher confirms and transactions

mirrored queues支持publisher confirms & transactions。

在transactions的情况下，只有当transactions被apply到queue的所有的mirrors，`tx-commit-ok`才会被返回给client。

在publisher confirm的情况下，只有当message被所有的mirrors接收到的情况下才会被confirm给publisher。

# flow control

rabbitmq使用了credit-based的算法来limit消息发布的速率。当publishers从queue所有的mirrors收到credit的时候，就可以发布消息。如果mirrors发行credit失败的时候，publisher就会拖延。在所有的mirror发行credit之前或者publisher会被block

# master failures & consumer cancellation

有些client可能会想要知道mirrored queue什么时候发生了fail over，这个时候可以设置`x-cancel-on-ha-failover`为true，这样当failover的时候，consumer cancellation notification就会被发送给consumer。然后consumer需要自己reissue `basic.consume`来重新consuming。

# unsynchronised mirrors

在任何时候都可以把一个node加入到cluster，根据配置，当node加进去的时候，就有可能在新的node上增加mirror。在这个时候，新的mirror是空的，不会包含queue中的已经存在的消息。而当新消息发布到queue的时候，mirror也会收到，慢慢地，当queue中的消息慢慢被consume掉之后，新mirror中的内容跟master里的完全一致的时候，就认为是fully synchronised了。

新加入的mirror并没有提供额外的冗余和可靠性保障，除非它被显示地同步（默认情况下ha-sync-mode=manual，不会主动同步消息到新节点）。而显示同步的时候，queue变得unresponsive，就没办法响应了，所以不建议在active的queue上做显示的同步，而只同步非active的queues。

当自动queue mirroring被enable，需要考虑queue涉及到的disk data set，拥有大数据集的queue（10g或者更多）需要复制到新加的mirror的时候，可能会造成network bandwidth和disk i/o的压力。

查看mirros的状态，是否被synchronised

```shell
[root@ci81hf1cmp001 ~]# rabbitmqctl list_queues name slave_pids synchronised_slave_pids
Timeout: 60.0 seconds ...
Listing queues for vhost / ...
name	slave_pids	synchronised_slave_pids
q-agent-notifier-port-update.ci81hf1cmp009.qa.webex.com	[<rabbit@ci81hf1cmp003.2.7061.2>, <rabbit@ci81hf1cmp002.2.7425.2>]	[<rabbit@ci81hf1cmp003.2.7061.2>, <rabbit@ci81hf1cmp002.2.7425.2>]
q-agent-notifier-security_group-update.ci81hf1cmp007.qa.webex.com	[<rabbit@ci81hf1cmp003.2.6524.2>, <rabbit@ci81hf1cmp002.2.6900.2>]	[<rabbit@ci81hf1cmp003.2.6524.2>, <rabbit@ci81hf1cmp002.2.6900.2>]
```

可以手工synchronise一个queue

```shell
rabbitmqctl sync_queue {name}
```

cancel一个正在进行中的synchronisation

```shell
rabbitmqctl cancel_sync_queue {name}
```

# Promotion of Unsynchronised Mirrors on Failure

默认情况下，如果一个queue的master挂了，或者失去连接了，或者从cluster removed了，最旧的mirror就会被promoted成新的master。在某些情况下，这个mirror可能会是unsynchronised，这样就会造成数据丢失。

从rabbitmq 3.7.5开始，`ha-promote-on-failure`可以用来配置是否能够promote unsynchronised的mirror为master。如果配置成`when-synced`，那么unsynchronised mirror就不会被promoted。默认是`always`，就是都会被promoted。配置成`when-synced`需要很谨慎，这会很依赖master的稳定性，而很多时候，queue的可用性比一致性来的更重要。如果配成`when-synced`，就意味着，如果没有synced mirror，就只能等master恢复才行了，如果master回不来，那只能删除queue或者重新定义queue了。而删除queue就丢失了所有的消息，这个损失是更大的。

# stopping nodes and synchronisation

如果包含了master的node被停掉，一些mirror会被promoted成master。如果继续停掉node，到了没有更多的mirror的时候：只存在一个node，就是新的master。如果mirrored queue被定义为durable，那么假如最后的node也被shutdown了，当这个最后的node再起来的时候durable messages还会保留。通常，如果你重启的是其他的node，而他们又是mirrored queue的一部分，他们会rejoin mirrored queue。

而对于一个Mirror来讲，当他rejoin到cluster的时候，它没办法知道自己queue的内容是不是跟master一致（比如发生了network partition），所以，当mirror rejoin到mirrored queue的时候，就会丢弃所有的durable的本地内容，然后清空queue中的内容，就跟新node加入cluster的是一样的

# Stopping Master Nodes with Only Unsynchronised Mirrors

如果所有的mirrors都还是unsynchronised的时候把master shutdown：

* 默认情况下（`ha-promote-on-shutdown`默认配置为`when-synced`），如果master因为主动的原因停掉，比如是通过rabbitmqctl stop命令停止或者优雅关闭OS，rabbitmq会拒绝premote unsynchronised mirror，为了避免message loss，所以这个时候queue是不可用的。
* 如果是因为异常原因被动地导致Master down，比如server或者node crash，网络outage等，会promote unsynchronised mirror。

如果希望在任何情况下都promote，可以把`ha-promote-on-shutdown`配置成`always`。

还有，在`ha-promote-on-failure`设置为`when-sync`的情况下，即使`ha-promote-on-shutdown`配置成`always`，unsynchronised的mirrors也不会被promoted。

* `ha-promote-on-failure`默认配置是`always`
* `ha-promote-on-shutdown`默认配置是`when-sync`

# Loss of a Master While All Mirrors are Stopped

当我们lost master的时候，很有可能所有mirror也都shutdown了。正常来讲，最后一个被shutdown的node会变成master，而我们也希望最后那个被shutdown的mirror成为新的master，因为最后被shutdown，就说明它保留了更多的Message，至少比其他mirror要多。

当我们执行`rabbitmqctl forget_cluster_node` 的时候，rabbitmq会为这个node上所有的master queue去尝试去找一个最近stopped mirror，然后把那个mirror promote成新的master。

rabbitmq只会在`forget_cluster_node`的时候promote stopped mirrors，因为每当一个mirror被重启的时候，会清空所有的message，所以我们需要在start mirror之前执行`rabbitmqctl forget_cluster_node` 

# batch synchronization

rabbitmq默认是批量的形式进行同步的，之前的版本默认是同步一条消息，现在默认是4096条消息，通过`ha-sync-batch-size`可以进行配置，这个值要怎么配需要考虑：

* 平均的message size
* network throughput
* net_tricktime

比如，` ha-sync-batch-size`设置为50000 message，每条message平均1kb，那么每次node间的sync大约在49M，需要确认network是可以处理这个流量的，如果花费了超过net_tricktime的时间来传送一个batch的message，会被认为发生了network partition。

很重要的一点是：当一个queue正在做同步的时候，所有其他的queue的operations都是会被blocked的，而由于多种因素，一个queue在同步的时候，可能会被block几分钟，几小时，甚至几天。关于同步的设置：

* `ha-sync-mode: manual`：默认配置，queue的mirror不会收已存在消息，只会接受新消息
* `ha-sync-mode: automatic`：当新的mirror加入的时候，会自动同步所有的消息

# References

[1] [https://www.rabbitmq.com/ha.html#batch-sync](https://www.rabbitmq.com/ha.html#batch-sync)

