---
layout: post
title:  "Nova compute node resource上报"
date:   2020-03-31 23:01:00 +0800
categories: Openstack
tags: Openstack-Nova
excerpt: Nova compute node resource上报
mathjax: true
typora-root-url: ../
---

# 问题起因

ci22tacmp188这个机器上，明明只有4个vm，都是4cpu 10gmem 80gdisk的

```shell
2020-03-30 08:11:37.781 513301 INFO nova.compute.resource_tracker [req-f44fe447-9168-4b02-8089-cfde06ebd807 - - - - -] Total usable vcpus: 48, total allocated vcpus: 24
2020-03-30 08:11:37.782 513301 INFO nova.compute.resource_tracker [req-f44fe447-9168-4b02-8089-cfde06ebd807 - - - - -] Final resource view: name=ci22tacmp188.webex.com phys_ram=392944MB used_ram=59392MB phys_disk=2637GB used_disk=400GB total_vcpus=48 used_vcpus=24 pci_stats=[]
```

理应是16cpu的，Nova却是上报了24个cpu

# 原理

```python
        instances = objects.InstanceList.get_by_host_and_node(
            context, self.host, self.nodename,
            expected_attrs=['system_metadata',
                            'numa_topology',
                            'flavor', 'migration_context'])

        # Now calculate usage based on instance utilization:
        self._update_usage_from_instances(context, instances)

        # Grab all in-progress migrations:
        migrations = objects.MigrationList.get_in_progress_by_host_and_node(
                context, self.host, self.nodename)

        self._pair_instances_to_migrations(migrations, instances)
        self._update_usage_from_migrations(context, migrations)

        # Detect and account for orphaned instances that may exist on the
        # hypervisor, but are not in the DB:
        orphans = self._find_orphaned_instances()
        self._update_usage_from_orphans(orphans)
```

看代码，resource_tracker计算当前compute节点资源的时候，是分了三部分：

* 本机的vm：InstanceList.get_by_host_and_node
* 正在进行中的migration：MigrationList.get_in_progress_by_host_and_node
* 孤儿instance：_find_orphaned_instances

本机的很好理解，孤儿不在本篇讨论范畴内，我们的问题出在计算正在进行中的migration list（用的是newton版本，比较老，最新版本其实已经fix这个问题）

```python
def migration_get_in_progress_by_host_and_node(context, host, node):

    return model_query(context, models.Migration).\
            filter(or_(and_(models.Migration.source_compute == host,
                            models.Migration.source_node == node),
                       and_(models.Migration.dest_compute == host,
                            models.Migration.dest_node == node))).\
            filter(~models.Migration.status.in_(['accepted', 'confirmed',
                                                 'reverted', 'error',
                                                 'failed', 'completed',
                                                 'cancelled'])).\
            options(joinedload_all('instance.system_metadata')).\
            all()
```

这里计算的时候，有一个filter，会过滤掉我们认为已经结束（不管是成功还是失败）了的（不在running中的migration），只统计正在进行中的migration。这是为了防止migration 成功或者失败，Dest跟source可能会overcommit的情况。

```shell
| 2506 | ci22tacmp189.webex.com | ci22tacmp188.webex.com | ci22tacmp189.webex.com | ci22tacmp188.webex.com | 10.240.200.151 | done          | 3548b1aa-1270-4448-9eb5-500ea79b5489 | None       | None       | 2019-03-27T05:02:59.000000 | 2019-03-27T05:03:19.000000 | evacuation     |
```

而我们数据库里有这样一个情况，曾经有一台vm被evacuation过，从189到了188，状态是done的。按照我们目前的代码，done状态不会被filter掉，所以就认为是进行中，导致不管是189还是188上，都会把这台vm的信息统计进去

```shell
bash-3.2$ nova show 3548b1aa-1270-4448-9eb5-500ea79b5489
+--------------------------------------+----------------------------------------------------------------------+
| Property                             | Value                                                                |
+--------------------------------------+----------------------------------------------------------------------+
| OS-DCF:diskConfig                    | MANUAL                                                               |
| OS-EXT-AZ:availability_zone          | ta-ssd-az1                                                           |
| OS-EXT-SRV-ATTR:host                 | ci22tacmp206.webex.com                                               |
| OS-EXT-SRV-ATTR:hostname             | vsta1mga005.webex.com                                                |
| OS-EXT-SRV-ATTR:hypervisor_hostname  | ci22tacmp206.webex.com                                               |
| OS-EXT-SRV-ATTR:instance_name        | instance-0002ab53                                                    |
| OS-EXT-SRV-ATTR:kernel_id            |                                                                      |
| OS-EXT-SRV-ATTR:launch_index         | 0                                                                    |
| OS-EXT-SRV-ATTR:ramdisk_id           |                                                                      |
| OS-EXT-SRV-ATTR:reservation_id       | r-3lq0us77                                                           |
| OS-EXT-SRV-ATTR:root_device_name     | /dev/vda                                                             |
| OS-EXT-SRV-ATTR:user_data            | -                                                                    |
| OS-EXT-STS:power_state               | 1                                                                    |
| OS-EXT-STS:task_state                | -                                                                    |
| OS-EXT-STS:vm_state                  | active                                                               |
| OS-SRV-USG:launched_at               | 2019-03-27T05:03:19.000000                                           |
| OS-SRV-USG:terminated_at             | -                                                                    |
| accessIPv4                           |                                                                      |
| accessIPv6                           |                                                                      |
| config_drive                         |                                                                      |
| created                              | 2019-01-23T07:24:12Z                                                 |
| description                          | -                                                                    |
| flavor                               | 8vcpu.16mem.80ssd.0eph (7848f9aa-e445-4729-b732-46d65b1e267d)        |
| hostId                               | 435d1d5e6ed2c9bf0c68e476b399bac418a586ebb2238fab8c662e70             |
```

加上这台的量，刚好就是我们目前上报的。

所以这里是nova的一个bug，查了下，果然社区已经fix了：

```shell
accepted,confirmed,reverted,error,failed,completed,cancelled -》我们的代码
confirmed,reverted,error,failed,completed,cancelled,done -》最新的代码
```

也就是done应该被认为是已完成，accepted应该被认为是进行中，分别在两个patch里被fix了

[https://github.com/openstack/nova/commit/299a5496f0979dbec8e0b6f25a58a282a55e8699](https://github.com/openstack/nova/commit/299a5496f0979dbec8e0b6f25a58a282a55e8699)
[https://github.com/openstack/nova/commit/961c2945491ebcea3cf1cb175a06d057155aa5a5](https://github.com/openstack/nova/commit/961c2945491ebcea3cf1cb175a06d057155aa5a5)

