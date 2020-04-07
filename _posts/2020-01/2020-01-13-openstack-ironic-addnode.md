---
layout: post
title:  "openstack standalone ironic添加node及ironic状态机"
date:   2020-01-13 23:00:00 +0800
categories: Openstack
tags: Openstack-Ironic
excerpt: standalone ironic环境增加node及ironic状态机
mathjax: true
---

# node create

rc文件

```shell
export OS_AUTH_TYPE=none
export OS_ENDPOINT=http://10.121.246.183:6385
```

增加node

```shell
[root@rain-standalone-ironic ~]# openstack baremetal node create --driver ipmi \
>     --driver-info ipmi_address=10.225.11.111 \
>     --driver-info ipmi_username=admin \
>     --driver-info ipmi_password=***** \
>     --driver-info deploy_kernel=file:///tftpboot/ironic-agent.kernel \
>     --driver-info deploy_ramdisk=file:///tftpboot/ironic-agent.initramfs
```

manage, provide

```shell
[root@rain-standalone-ironic ~]# openstack baremetal node list
+--------------------------------------+------+---------------+-------------+--------------------+-------------+
| UUID                                 | Name | Instance UUID | Power State | Provisioning State | Maintenance |
+--------------------------------------+------+---------------+-------------+--------------------+-------------+
| ab556e18-eacb-4947-aa7c-067cdd376b49 | None | None          | None        | enroll             | False       |
+--------------------------------------+------+---------------+-------------+--------------------+-------------+
[root@rain-standalone-ironic ~]# openstack baremetal node manage ab556e18-eacb-4947-aa7c-067cdd376b49
[root@rain-standalone-ironic ~]# openstack baremetal node list
+--------------------------------------+------+---------------+-------------+--------------------+-------------+
| UUID                                 | Name | Instance UUID | Power State | Provisioning State | Maintenance |
+--------------------------------------+------+---------------+-------------+--------------------+-------------+
| ab556e18-eacb-4947-aa7c-067cdd376b49 | None | None          | power on    | manageable         | False       |
+--------------------------------------+------+---------------+-------------+--------------------+-------------+
[root@rain-standalone-ironic ~]# openstack baremetal node provide ab556e18-eacb-4947-aa7c-067cdd376b49
[root@rain-standalone-ironic ~]# openstack baremetal node list
+--------------------------------------+------+---------------+-------------+--------------------+-------------+
| UUID                                 | Name | Instance UUID | Power State | Provisioning State | Maintenance |
+--------------------------------------+------+---------------+-------------+--------------------+-------------+
| ab556e18-eacb-4947-aa7c-067cdd376b49 | None | None          | power on    | available          | False       |
+--------------------------------------+------+---------------+-------------+--------------------+-------------+
```

# state machine

![Ironic state transitions](../assets/images/states.svg)

## enroll

openstack baremetal node create，添加node后，ironic就知道了node的存在，所以，就到了enroll状态

## manageable

openstack baremetal node manage，ironic根据提供的driver info，检查一下credential是不是正确，是不是可以连通baremetal node，如果可以连通，则变成manageable状态

## available

openstack baremetal node provide，node进行preconfigure和clean，然后变成available状态，这个状态表示ready to be provisioned

## active

当一个node被成功provisioned，有workload跑在上面的时候，就变成了active的状态

## error

删除一个active deployment失败的时候会进入error状态

## rescue

当有一个rescue ramdisk运行在node上的时候，进入rescue状态

# References

[1] [https://docs.openstack.org/ironic/latest/contributor/states.html](https://docs.openstack.org/ironic/latest/contributor/states.html)