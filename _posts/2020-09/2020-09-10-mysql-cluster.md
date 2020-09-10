---
layout: post
title:  "mysql db跑在hdd导致cluster变慢"
date:   2020-09-10 23:00:00 +0800
categories: Linux
tags: Linux-Service
excerpt: mysql db跑在hdd导致cluster变慢
mathjax: true
typora-root-url: ../
---

# mysql db跑在hdd导致cluster变慢

最近两个星期，产线ln环境会出现API很慢的情况，尤其keystone token issue，需要花费10-20s的时间

```shell
MINSU-M-M1RW:kolla-ansible minsu$ time openstack token issue
+------------+----------------------------------+
| Field | Value |
+------------+----------------------------------+
| expires | 2020-09-09T18:15:06+0000 |
| id | 540be0a3ab8f415fab1467104bcdb60d |
| project_id | 2a7d15ccfd2c448ba234ef1301685c27 |
| user_id | 8cf63a8b0dd94d8e8aebe1c2f639acec |
+------------+----------------------------------+

real 0m18.256s
user 0m0.993s
sys 0m0.907s
```

导致一些api失败

这时候可以看到mysql的一些状态

```mysql
ci22lncmp002

| wsrep_local_send_queue       | 0                                                           |
| wsrep_local_send_queue_avg   | 0.000000                                                    |
| wsrep_local_recv_queue       | 29                                                          |
| wsrep_local_recv_queue_avg   | 17.979120                                                   |
| wsrep_local_cached_downto    | 3567353361                                                  |
| wsrep_flow_control_paused_ns | 8044453386074846                                            |
| wsrep_flow_control_paused    | 0.306026                                                    |
| wsrep_flow_control_sent      | 52829969                                                    |
| wsrep_flow_control_recv      | 52875205                                                    |
ci22lncmp001

| wsrep_local_replays          | 0                                                           |
| wsrep_local_send_queue       | 746                                                         |
| wsrep_local_send_queue_avg   | 59.057416                                                   |
| wsrep_local_recv_queue       | 0                                                           |
| wsrep_local_recv_queue_avg   | 0.001056                                                    |
| wsrep_local_cached_downto    | 3567339300                                                  |
| wsrep_flow_control_paused_ns | 8047674478358211                                            |
| wsrep_flow_control_paused    | 0.305765                                                    |
| wsrep_flow_control_sent      | 1                                                           |
| wsrep_flow_control_recv      | 52893698
```

关注wsrep_local_send_queue跟wsrep_local_recv_queue，一直都是非0的状态 

wsrep_local_send_queue：发送队列的长度 The number of mysql writesets waiting to be sent

wsrep_local_recv_queue：接收队列的长度 The number of mysql writesets waiting to be applied

这两个值 > 0.0意味着复制过程被节流了

后来我们发现是因为我们的mysql数据（/var/lib/mysql）跑在hdd上了，hdd的读写速度跟ssd比差了一大截，明显不能满足mysql的需求，后来把数据库移到ssd上后，问题得到了解决

```shell
MINSU-M-M1RW:kolla-ansible minsu$ time openstack token issue
+------------+----------------------------------+
| Field | Value |
+------------+----------------------------------+
| expires | 2020-09-09T17:58:19+0000 |
| id | 494103a99832498abb208f04e4fedf1d |
| project_id | 2a7d15ccfd2c448ba234ef1301685c27 |
| user_id | 8cf63a8b0dd94d8e8aebe1c2f639acec |
+------------+----------------------------------+

real 0m3.440s
user 0m0.799s
sys 0m0.512s
```



