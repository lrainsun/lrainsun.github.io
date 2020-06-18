---
layout: post
title:  "rabbitmq monitoring -- Individual Node Checks"
date:   2020-06-18 23:00:00 +0800
categories: Linux
tags: Linux-Service
excerpt: rabbitmq monitoring -- Individual Node Checks
mathjax: true
typora-root-url: ../
---

# rabbitmq monitoring

rabbitmq的问题现在虽然暂时fix了，cluster正常，但我们其实没有完全理解原理，这几天都在看rabbitmq，包括mirror的机制啥的，今天看了rabbitmq monitoring，不管是预警还是事后分析，monitoring都还是很重要的。

很多工具都可以用来监控，prometheus+grafana是最推荐的一种方式。rabbitmq 3.8开始已经内置了对prometheus的支持

- [What is monitoring](https://www.rabbitmq.com/monitoring.html#approaches-to-monitoring), what common approaches to it exist and why it is important.
- Built-in and [external monitoring](https://www.rabbitmq.com/monitoring.html#external-monitoring) options
- What [infrastructure and kernel metrics](https://www.rabbitmq.com/monitoring.html#system-metrics) are important to monitor
- What RabbitMQ metrics are available:
  - [Node metrics](https://www.rabbitmq.com/monitoring.html#node-metrics)
  - [Queue metrics](https://www.rabbitmq.com/monitoring.html#queue-metrics)
  - [Cluster-wide metrics](https://www.rabbitmq.com/monitoring.html#cluster-wide-metrics)
- [How frequently](https://www.rabbitmq.com/monitoring.html#monitoring-frequency) should monitoring checks be performed?
- [Application-level metrics](https://www.rabbitmq.com/monitoring.html#app-metrics)
- How to approach [node health checking](https://www.rabbitmq.com/monitoring.html#health-checks) and why it's more involved than a single CLI command.
- [Log aggregation](https://www.rabbitmq.com/monitoring.html#log-aggregation)
- [Command-line based observer](https://www.rabbitmq.com/monitoring.html#diagnostics-observer) tool

# 系统和rabbitmq metrics

metrics分成两类，system & rabbitmq

## rabbitmq-specific

they are [collected and reported by RabbitMQ nodes](https://www.rabbitmq.com/monitoring.html#rabbitmq-metrics). In this guide we refer to them as "RabbitMQ metrics". Examples include the number of socket descriptors used, total number of enqueued messages or inter-node communication traffic rates. 

[Management UI and External Monitoring Systems](https://www.rabbitmq.com/monitoring.html#external-monitoring)

The RabbitMQ [management plugin](https://www.rabbitmq.com/management.html) provides an API for accessing RabbitMQ metrics. The plugin will store up to one day's worth of metric data. Longer term monitoring should be accomplished with an [external tool](https://www.rabbitmq.com/monitoring.html#monitoring-tools).

## system

[Infrastructure and Kernel Metrics](https://www.rabbitmq.com/monitoring.html#system-metrics)

Others metrics are [collected and reported by the OS kernel](https://www.rabbitmq.com/monitoring.html#system-metrics). Such metrics are often called system metrics or infrastructure metrics. System metrics are not specific to RabbitMQ. Examples include CPU utilisation rate, amount of memory used by processes, network packet loss rate, et cetera. Both types are important to track. Individual metrics are not always useful but when analysed together, they can provide a more complete insight into the state of the system. Then operators can form a hypothesis about what's going on and needs addressing.

* CPU stats (user, system, iowait & idle percentages)
* Memory usage (used, buffered, cached & free percentages)
* [Virtual Memory](https://www.kernel.org/doc/Documentation/sysctl/vm.txt) statistics (dirty page flushes, writeback volume)
* Disk I/O (operations & amount of data transferred per unit time, time to service operations)
* Free disk space on the mount used for the [node data directory](https://www.rabbitmq.com/relocate.html)
* File descriptors used by beam.smp vs. [max system limit](https://www.rabbitmq.com/networking.html#open-file-handle-limit)
* TCP connections by state (ESTABLISHED, CLOSE_WAIT, TIME_WAIT)
* Network throughput (bytes received, bytes sent) & maximum network throughput)
* Network latency (between all RabbitMQ nodes in a cluster as well as to/from clients)

## 单node check

有很多方面的monitoring，今天主要看了[Individual Node Checks](https://www.rabbitmq.com/monitoring.html#individual-checks)，分成好几个stage，越高的stage check更复杂

### stage1

确认runtime是在运行的，cli可以用它进行身份认证

```shell
[root@ci81hf1cmp003 ~]# rabbitmq-diagnostics -q ping

Ping succeeded
```

升级和维护的时候可能会误报

### stage2

增加了一些获取基本系统信息的check

```shell
[root@ci81hf1cmp002 ~]# rabbitmq-diagnostics -q status
[{pid,437303},
 {running_applications,
     [{rabbitmq_management,"RabbitMQ Management Console","3.7.21"},
      {amqp_client,"RabbitMQ AMQP Client","3.7.21"},
      {rabbitmq_web_dispatch,"RabbitMQ Web Dispatcher","3.7.21"},
      {rabbitmq_management_agent,"RabbitMQ Management Agent","3.7.21"},
      {rabbit,"RabbitMQ","3.7.21"},
      {os_mon,"CPO  CXC 138 46","2.5.1"},
      {rabbit_common,
          "Modules shared by rabbitmq-server and rabbitmq-erlang-client",
          "3.7.21"},
      {cowboy,"Small, fast, modern HTTP server.","2.6.1"},
      {mnesia,"MNESIA  CXC 138 12","4.16.1"},
      {cowlib,"Support library for manipulating Web protocols.","2.7.0"},
      {sysmon_handler,"Rate-limiting system_monitor event handler","1.1.0"},
      {lager,"Erlang logging framework","3.8.0"},
      {ranch,"Socket acceptor pool for TCP protocols.","1.7.1"},
      {ssl,"Erlang/OTP SSL application","9.4"},
      {public_key,"Public key infrastructure","1.7"},
      {asn1,"The Erlang ASN1 compiler version 5.0.9","5.0.9"},
      {inets,"INETS  CXC 138 49","7.1.1"},
      {tools,"DEVTOOLS  CXC 138 16","3.2.1"},
      {jsx,"a streaming, evented json parsing toolkit","2.9.0"},
      {observer_cli,"Visualize Erlang Nodes On The Command Line","1.5.2"},
      {recon,"Diagnostic tools for production use","2.5.0"},
      {stdout_formatter,
          "Tools to format paragraphs, lists and tables as plain text",
          "0.2.2"},
      {credentials_obfuscation,
          "Helper library that obfuscates sensitive values in process state",
          "1.1.0"},
      {crypto,"CRYPTO","4.6.2"},
      {xmerl,"XML parser","1.3.22"},
      {goldrush,"Erlang event stream processor","0.1.9"},
      {compiler,"ERTS  CXC 138 10","7.4.8"},
      {syntax_tools,"Syntax tools","2.2.1"},
      {sasl,"SASL  CXC 138 11","3.4.1"},
      {stdlib,"ERTS  CXC 138 10","3.10"},
      {kernel,"ERTS  CXC 138 10","6.5"}]},
 {os,{unix,linux}},
 {erlang_version,
     "Erlang/OTP 22 [erts-10.5.4] [source] [64-bit] [smp:32:32] [ds:32:32:10] [async-threads:128] [hipe]\n"},
 {memory,
     [{connection_readers,18824600},
      {connection_writers,1048524},
      {connection_channels,6756840},
      {connection_other,49696412},
      {queue_procs,11068256},
      {queue_slave_procs,6201292},
      {plugins,47475732},
      {other_proc,20968052},
      {metrics,4378204},
      {mgmt_db,26254352},
      {mnesia,2812080},
      {other_ets,4206800},
      {binary,648012888},
      {msg_index,40064},
      {code,24533767},
      {atom,1468577},
      {other_system,32021288},
      {allocated_unused,278959312},
      {reserved_unallocated,0},
      {strategy,rss},
      {total,[{erlang,905767728},{rss,498446336},{allocated,1184727040}]}]},
 {alarms,[]},
 {listeners,
     [{clustering,35672,"::"},{amqp,5672,"0.0.0.0"},{http,15672,"0.0.0.0"}]},
 {vm_memory_calculation_strategy,rss},
 {vm_memory_high_watermark,0.4},
 {vm_memory_limit,53995508531},
 {disk_free_limit,50000000},
 {disk_free,104766648320},
 {file_descriptors,
     [{total_limit,63900},
      {total_used,644},
      {sockets_limit,57508},
      {sockets_used,637}]},
 {processes,[{limit,1048576},{used,9202}]},
 {run_queue,0},
 {uptime,46627},
 {kernel,{net_ticktime,60}}]
```

### stage3

增加了rabbitmq application running的check（没有被rabbitmqctl stop_app停掉或者也没有network partition，以及resource告警）

```shell
[root@ci81hf1cmp001 ~]# rabbitmq-diagnostics -q alarms
Node rabbit@ci81hf1cmp001 reported no alarms, local or clusterwide
[root@ci81hf1cmp001 ~]# rabbitmq-diagnostics check_running

Checking if RabbitMQ is running on node rabbit@ci81hf1cmp001 ...
RabbitMQ on node rabbit@ci81hf1cmp001 is fully booted and running

[root@ci81hf1cmp001 ~]# rabbitmq-diagnostics check_local_alarms
Asking node rabbit@ci81hf1cmp001 to report any local resource alarms ...
Node rabbit@ci81hf1cmp001 reported no local alarms

[root@ci81hf1cmp001 ~]# rabbitmq-diagnostics -q check_running && rabbitmq-diagnostics -q check_local_alarms

RabbitMQ on node rabbit@ci81hf1cmp001 is fully booted and running
Node rabbit@ci81hf1cmp001 reported no local alarms
```

误报可能性比较小，在high runtime memory watermark周围的时候会有大概率会误报，升级和维护的时候可能会误报

#### 内存检查

https://www.rabbitmq.com/memory-use.html

可以通过api获取内存使用

```shell
[root@ci81hf1cmp001 ~]# curl -u rabbituser:password -X GET http://127.0.0.1:15672/api/nodes/rabbit@ci81hf1cmp001/memory | jq .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   527  100   527    0     0   2490      0 --:--:-- --:--:-- --:--:--  2497
{
  "memory": {
    "total": {
      "allocated": 1181827072,
      "rss": 455733248,
      "erlang": 872283312
    },
    "strategy": "rss",
    "reserved_unallocated": 0,
    "allocated_unused": 309543760,
    "other_system": 32415964,
    "other_proc": 35862584,
    "plugins": 38882096,
    "queue_slave_procs": 3130608,
    "queue_procs": 16618908,
    "connection_other": 53308468,
    "connection_channels": 10589536,
    "connection_writers": 1452556,
    "connection_readers": 19314984,
    "metrics": 5322572,
    "mgmt_db": 19267728,
    "mnesia": 2843880,
    "other_ets": 4321696,
    "binary": 602385408,
    "msg_index": 40016,
    "code": 25049539,
    "atom": 1476769
  }
}

[root@ci81hf1cmp001 ~]# curl -u rabbituser:password -X GET http://127.0.0.1:15672/api/nodes/rabbit@ci81hf1cmp001/memory | jq ".memory.total.allocated"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   526  100   526    0     0   2378      0 --:--:-- --:--:-- --:--:--  2369
1188327424
```

运维人员需要知道一个node的memory use，需要知道谁用了绝大多数的内存

```shell
[root@ci81hf1cmp001 ~]# rabbitmq-diagnostics memory_breakdown
Reporting memory breakdown on node rabbit@ci81hf1cmp001...
binary: 0.6018 gb (50.82%)
allocated_unused: 0.325 gb (27.45%)
connection_other: 0.0525 gb (4.43%)
other_proc: 0.0355 gb (3.0%)
plugins: 0.0328 gb (2.77%)
other_system: 0.0324 gb (2.74%)
code: 0.025 gb (2.12%)
connection_readers: 0.0192 gb (1.63%)
mgmt_db: 0.019 gb (1.6%)
queue_procs: 0.0155 gb (1.31%)
connection_channels: 0.0076 gb (0.64%)
metrics: 0.0053 gb (0.45%)
other_ets: 0.0043 gb (0.37%)
mnesia: 0.0028 gb (0.24%)
queue_slave_procs: 0.0026 gb (0.22%)
atom: 0.0015 gb (0.12%)
connection_writers: 0.0011 gb (0.09%)
msg_index: 0.0 gb (0.0%)
reserved_unallocated: 0.0 gb (0.0%)
```

##### 还可以用rabbitmq-diagnostics observer（跟top类似）查看每个runtime processes的详细信息，包括

- Runtime version information
- CPU and schedule stats
- Memory allocation and usage stats
- Top processes by CPU (reductions) and memory usage
- Network link stats
- Detailed process information such as basic TCP socket stats

##### 总内存使用量计算策略

rabbitmq可以使用不同的策略来计算一个node使用了多少内存（用vm_memory_calculation_strategy配置）

* rss: uses OS-specific means of querying the kernel to find RSS (Resident Set Size) value of the node OS process. 
* allocated：queries runtime memory allocator information

Introduced in 3.6.11. `rss` is the default as of 3.6.12

会影响memory allocator information

##### rabbitmq-top

还可以enable rabbitmq-top来helps identify runtime processes ("lightweight threads") that consume most memory or scheduler (CPU) time.

```shell
[root@ci81hf1cmp003 ~]# rabbitmq-plugins enable rabbitmq_top
Enabling plugins on node rabbit@ci81hf1cmp003:
rabbitmq_top
The following plugins have been configured:
  rabbitmq_management
  rabbitmq_management_agent
  rabbitmq_top
  rabbitmq_web_dispatch
Applying plugin configuration to rabbit@ci81hf1cmp003...
The following plugins have been enabled:
  rabbitmq_top

[root@ci81hf1cmp003 ~]# service rabbitmq-server restart
Redirecting to /bin/systemctl restart rabbitmq-server.service
```

enable后需要restart service

![img](/../assets/images/58113632.png)

- Memory used
- Reductions (unit of scheduler/CPU consumption)
- Erlang mailbox length
- For gen_server2 processes, internal operation buffer length

![img](/../assets/images/58298820.png)

ETS (internal key/value store) tables. The tables can be sorted by the amount of memory used or number of rows

##### 一条消息需要用多少内存？

A message has multiple parts that use up memory:

- Payload: >= 1 byte, variable size, typically few hundred bytes to a few hundred kilobytes
- Protocol attributes: >= 0 bytes, variable size, contains headers, priority, timestamp, reply to, etc.
- RabbitMQ metadata: >= 720 bytes, variable size, contains exchange, routing keys, message properties, persistence, redelivery status, etc.
- RabbitMQ message ordering structure: 16 bytes

Messages with a 1KB payload will use up 2KB of memory once attributes and metadata is factored in.

Some messages can be stored on disk, but still have their metadata kept in memory.

##### 一个queue需要多少内存？

每个queue背后是erlang进程，如果queue是mirrored，每个mirror也是分开的erlang进程

queue的master是一个single erlang process，所以消息的ordering可以保证，多个queue是多个erlang进程，可以获取到公平的cpu时间，所以没有queue会block其他queue

```shell
[root@ci81hf1cmp001 ~]# curl -u rabbituser:password -X GET http://127.0.0.1:15672/api/queues/%2f/l3_agent | python -m json.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2697  100  2697    0     0   270k      0 --:--:-- --:--:-- --:--:--  292k
{
    "arguments": {
        "x-ha-policy": "all"
    },
    "auto_delete": false,
    "backing_queue_status": {
        "avg_ack_egress_rate": 0.0,
        "avg_ack_ingress_rate": 0.0,
        "avg_egress_rate": 0.0,
        "avg_ingress_rate": 0.0,
        "delta": [
            "delta",
            "undefined",
            0,
            0,
            "undefined"
        ],
        "len": 0,
        "mirror_seen": 0,
        "mirror_senders": 0,
        "mode": "default",
        "next_seq_id": 0,
        "q1": 0,
        "q2": 0,
        "q3": 0,
        "q4": 0,
        "target_ram_count": "infinity"
    },
    "consumer_details": [
        {
            "ack_required": true,
            "arguments": {},
            "channel_details": {
                "connection_name": "10.225.18.11:53944 -> 10.225.18.12:5672",
                "name": "10.225.18.11:53944 -> 10.225.18.12:5672 (1)",
                "node": "rabbit@ci81hf1cmp002",
                "number": 1,
                "peer_host": "10.225.18.11",
                "peer_port": 53944,
                "user": "rabbituser"
            },
            "consumer_tag": "1",
            "exclusive": false,
            "prefetch_count": 0,
            "queue": {
                "name": "l3_agent",
                "vhost": "/"
            }
        },
        {
            "ack_required": true,
            "arguments": {},
            "channel_details": {
                "connection_name": "10.225.18.12:43632 -> 10.225.18.12:5672",
                "name": "10.225.18.12:43632 -> 10.225.18.12:5672 (1)",
                "node": "rabbit@ci81hf1cmp002",
                "number": 1,
                "peer_host": "10.225.18.12",
                "peer_port": 43632,
                "user": "rabbituser"
            },
            "consumer_tag": "1",
            "exclusive": false,
            "prefetch_count": 0,
            "queue": {
                "name": "l3_agent",
                "vhost": "/"
            }
        },
        {
            "ack_required": true,
            "arguments": {},
            "channel_details": {
                "connection_name": "10.225.18.13:34108 -> 10.225.18.12:5672",
                "name": "10.225.18.13:34108 -> 10.225.18.12:5672 (1)",
                "node": "rabbit@ci81hf1cmp002",
                "number": 1,
                "peer_host": "10.225.18.13",
                "peer_port": 34108,
                "user": "rabbituser"
            },
            "consumer_tag": "1",
            "exclusive": false,
            "prefetch_count": 0,
            "queue": {
                "name": "l3_agent",
                "vhost": "/"
            }
        }
    ],
    "consumer_utilisation": null,
    "consumers": 3,
    "deliveries": [],
    "durable": false,
    "effective_policy_definition": {
        "ha-mode": "all",
        "ha-sync-mode": "automatic"
    },
    "exclusive": false,
    "exclusive_consumer_tag": null,
    "garbage_collection": {
        "fullsweep_after": 65535,
        "max_heap_size": 0,
        "min_bin_vheap_size": 46422,
        "min_heap_size": 233,
        "minor_gcs": 59
    },
    "head_message_timestamp": null,
    "idle_since": "2020-06-18 8:29:02",
    "incoming": [],
    "memory": 12772,
    "message_bytes": 0,
    "message_bytes_paged_out": 0,
    "message_bytes_persistent": 0,
    "message_bytes_ram": 0,
    "message_bytes_ready": 0,
    "message_bytes_unacknowledged": 0,
    "messages": 0,
    "messages_details": {
        "rate": 0.0
    },
    "messages_paged_out": 0,
    "messages_persistent": 0,
    "messages_ram": 0,
    "messages_ready": 0,
    "messages_ready_details": {
        "rate": 0.0
    },
    "messages_ready_ram": 0,
    "messages_unacknowledged": 0,
    "messages_unacknowledged_details": {
        "rate": 0.0
    },
    "messages_unacknowledged_ram": 0,
    "name": "l3_agent",
    "node": "rabbit@ci81hf1cmp002",
    "operator_policy": null,
    "policy": "ha-all",
    "recoverable_slaves": null,
    "reductions": 57529,
    "reductions_details": {
        "rate": 0.0
    },
    "slave_nodes": [
        "rabbit@ci81hf1cmp001",
        "rabbit@ci81hf1cmp003"
    ],
    "state": "running",
    "synchronised_slave_nodes": [
        "rabbit@ci81hf1cmp003",
        "rabbit@ci81hf1cmp001"
    ],
    "vhost": "/"
}
```

* memory: memory used by the queue process, accounts message metadata (at least 720 bytes per message), does not account for message payloads over 64 bytes
* message_bytes_ram: memory used by the message payloads, regardless of the size

If messages are small, message metadata can use more memory than the message payload. 10,000 messages with 1 byte of payload will use 10KB of message_bytes_ram (payload) & 7MB of memory (metadata).

If message payloads are large, they will not be reflected in the queue process memory. 10,000 messages with 100 KB of payload will use 976MB of message_bytes_ram (payload) & 7MB of memory (metadata).

##### 为什么publishing/consuming的时候，queue的memory query会增加和减少呢？

erlang为每个erlang进程都有gc，per queue的gc

gc run的时候，在deallocationg unused memory之前会拷贝used process memory，会造成queue process using up to 2倍的memory

### stage4

增加了listener的检查（通过临时tcp connection）

```shell
[root@ci81hf1cmp001 ~]# rabbitmq-diagnostics listeners
Asking node rabbit@ci81hf1cmp001 to report its protocol listeners ...Interface: [::], port: 35672, protocol: clustering, purpose: inter-node and CLI tool communicationInterface: [::], port: 5672, protocol: amqp, purpose: AMQP 0-9-1 and AMQP 1.0Interface: [::], port: 15672, protocol: http, purpose: HTTP API

[root@ci81hf1cmp001 ~]# netstat -natp|grep 35672
tcp 0 0 [0.0.0.0:35672](http://0.0.0.0:35672)0.0.0.0:* LISTEN 92261/beam.smp
tcp 0 0 [10.225.18.11:45866](http://10.225.18.11:45866)[10.225.18.13:35672](http://10.225.18.13:35672)ESTABLISHED 92261/beam.smp
tcp 0 0 [10.225.18.11:47014](http://10.225.18.11:47014)[10.225.18.12:35672](http://10.225.18.12:35672)ESTABLISHED 92261/beam.smp

[root@ci81hf1cmp001 ~]# rabbitmq-diagnostics check_port_connectivity
Testing TCP connections to all active listeners on node rabbit@ci81hf1cmp001 ...
Successfully connected to ports 5672, 15672, 35672 on node rabbit@ci81hf1cmp001.
```

### stage5

增加是否有failed virtual hosts的检查

```shell
[root@ci81hf1cmp001 ~]# rabbitmq-diagnostics check_virtual_hosts
Checking if all vhosts are running on node rabbit@ci81hf1cmp001 ...
Node rabbit@ci81hf1cmp001 reported all vhosts as running
```

### stage6

增加了channel和queue process在目标node是否存活的检查

```shell
[root@ci81hf1cmp001 ~]# rabbitmq-diagnostics -q check_port_connectivity && \
> rabbitmq-diagnostics -q node_health_check
Successfully connected to ports 5672, 15672, 35672 on node rabbit@ci81hf1cmp001.
Health check passed
```

如果有很多queue和channel，这是一个**expensive operation**，在高系统负载之下容易误报

### optional check 1

检查enabled plugins

```shell
[root@ci81hf1cmp001 ~]# rabbitmq-plugins -q list --enabled --minimal
rabbitmq_management
rabbitmq_top

[root@ci81hf1cmp001 ~]# rabbitmq-plugins -q list --enabled
 Configured: E = explicitly enabled; e = implicitly enabled
 | Status: * = running on rabbit@ci81hf1cmp001
 |/
[E*] rabbitmq_management 3.7.26
[E ] rabbitmq_top        3.7.26
```

# References

[1] [https://www.rabbitmq.com/monitoring.html](https://www.rabbitmq.com/monitoring.html)