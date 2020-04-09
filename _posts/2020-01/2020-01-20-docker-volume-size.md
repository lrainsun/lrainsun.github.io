---
layout: post
title:  "docker volume大小的修改"
date:   2020-01-20 18:00:00 +0800
categories: Docker
tags: Docker-Volume
excerpt: docker volume大小的修改
mathjax: true
---

# 题外话

昨天把standalone ironic node添加进去啦，昨晚11点多大神告知Dino老师把vlan配好，dhcp通了，我正看着《三体》呢。anyway，正式开始调试boot process，然后打脸了，第一步就失败了

```python
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils [req-1eab3bba-da3e-4479-a7d5-c656ff9a4ccd - - - - -] Unexpected error while preparing to deploy to node ab556e18-eacb-4947-aa7c-067cdd376b49: MissingAuthPlugin: An auth plugin is required to determine endpoint URL
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils Traceback (most recent call last):
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils   File "/usr/lib/python2.7/site-packages/ironic/conductor/manager.py", line 3758, in do_node_deploy
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils     task.driver.deploy.prepare(task)
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils   File "/usr/lib/python2.7/site-packages/ironic_lib/metrics.py", line 60, in wrapped
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils     result = f(*args, **kwargs)
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils   File "/usr/lib/python2.7/site-packages/ironic/conductor/task_manager.py", line 148, in wrapper
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils     return f(*args, **kwargs)
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils   File "/usr/lib/python2.7/site-packages/ironic/drivers/modules/iscsi_deploy.py", line 529, in prepare
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils     task.driver.network.add_provisioning_network(task)
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils   File "/usr/lib/python2.7/site-packages/ironic/drivers/modules/network/flat.py", line 102, in add_provisioning_network
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils     self._bind_flat_ports(task)
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils   File "/usr/lib/python2.7/site-packages/ironic/drivers/modules/network/flat.py", line 58, in _bind_flat_ports
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils     client = neutron.get_client(context=task.context)
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils   File "/usr/lib/python2.7/site-packages/ironic/common/neutron.py", line 68, in get_client
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils     auth=service_auth)
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils   File "/usr/lib/python2.7/site-packages/ironic/common/keystone.py", line 116, in get_endpoint
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils     result = get_adapter(group, **adapter_kwargs).get_endpoint()
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils   File "/usr/lib/python2.7/site-packages/keystoneauth1/adapter.py", line 282, in get_endpoint
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils     return self.session.get_endpoint(auth or self.auth, **kwargs)
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils   File "/usr/lib/python2.7/site-packages/keystoneauth1/session.py", line 1198, in get_endpoint
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils     auth = self._auth_required(auth, 'determine endpoint URL')
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils   File "/usr/lib/python2.7/site-packages/keystoneauth1/session.py", line 1138, in _auth_required
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils     raise exceptions.MissingAuthPlugin(msg_fmt % msg)
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils MissingAuthPlugin: An auth plugin is required to determine endpoint URL
2020-01-19 15:40:58.006 6 ERROR ironic.conductor.utils
```

这里明明add_provision_network调用了neutron client去create flat network。原来是认为dhcp_provider=None就表示这里是no op，看起来不是这样的。所以昨天的blog是有问题哒，所以光靠脑子想的事不靠谱啊，实践出真知。

# volume size

今天又遇到一个问题，在下载image到tftp对应目录的时候提示，空间不够了

```shell
2020-01-20 03:07:09.204 6 ERROR ironic.conductor.task_manager [req-be4631a4-83f3-4b68-8afc-48b91ac13d4d - - - - -] Node ab556e18-eacb-4947-aa7c-067cdd376b49 moved to provision state "deploy failed" from state "deploying"; target provision state is "active": InsufficientDiskSpace: Disk volume where '/var/lib/ironic/master_images/tmpmGWW6o' is located doesn't have enough disk space. Required 10240 MiB, only 3266 MiB available space present.
```

因为tftp server是ironic_pxe负责启动的，而数据是一个docker volume，貌似默认的volume size是10G，这样就不够大了。就需要修改docker volume size，最好是配置docker volume默认size。google一下

# 修改docker volume size

查到说这样可以配置:

> --storage-opt dm.basesize=20G

可是，试了一下，报错。我们的storage driver是overlay2，所以不支持dm.basesize

```shell
[graphdriver] prior storage driver overlay2 failed: overlay2: unknown option dm.basesize
failed to start daemon: error initializing graphdriver: overlay2: unknown option dm.basesize
```

再google一下:

##### `overlay2.size`

Sets the default max size of the container. It is supported only when the backing fs is `xfs` and mounted with `pquota` mount option. Under these conditions the user can pass any size less then the backing fs size.

###### Example

```shell
$ sudo dockerd -s overlay2 --storage-opt overlay2.size=1G
```

试一下加，限制最大容量

> --storage-opt overlay2.size=20G 

再次报错：

```shell
[graphdriver] prior storage driver overlay2 failed: Storage option overlay2.size not supported. Filesystem does not support Project Quota: Filesystem does not support, or has not enabled quotas
failed to start daemon: error initializing graphdriver: Storage option overlay2.size not supported. Filesystem does not support Project Quota: Filesystem does not support, or has not enabled quotas
```

意思是，需要enable quotas，这需要修改fstab

docker的overlay2需要的是`pquota`，在`/etc/fstab`中设置：

```shell
/dev/vdb /data xfs rw,pquota 0 0
```

而我们的volume在/var下，然后我发现/var本身的size就不够，所以这个方法还没尝试，后面如果试了在更新。

后来找了workaround的是。。换了个比较小的Image...

# References

[1] [https://blog.phpgao.com/docker_base_device_size.html](https://blog.phpgao.com/docker_base_device_size.html)

[2] [https://docs.docker.com/engine/reference/commandline/dockerd/#options-per-storage-driver](https://docs.docker.com/engine/reference/commandline/dockerd/#options-per-storage-driver)

[3] [https://www.lijiaocn.com/%E9%97%AE%E9%A2%98/2018/12/26/docker-overlay2-size-limit.html](https://www.lijiaocn.com/问题/2018/12/26/docker-overlay2-size-limit.html)