---
layout: post
title:  "openstack standalone ironic Networking service"
date:   2020-01-19 13:00:00 +0800
categories: Openstack
tags: Openstack-Ironic
excerpt: openstack standalone ironic Networking service
mathjax: true
---

# network interface in ironic

我们知道，当ironic跟openstack是集成在一起的时候，network service这块是neutron负责的，neutron会从他管理的网络里面为baremetal node分配deploy的时候使用的provision network，在deploy完成后会切换到tenant network。

T版本中默认enable了flat & noop

```python
enabled_network_interfaces     = ['flat', 'noop']
```

这个配置选项是在ironic.conf中可以修改

```python
enabled_network_interfaces = noop
```

当注册一个node的时候，有一个参数是Network_interface

这个地方如果是flat的话

> | network_interface | flat

会调用/usr/lib/python2.7/site-packages/ironic/drivers/modules/network/flat.py的实现。

比如add provision network的时候，会去调用neutron flat的接口

```python
def add_provisioning_network(self, task):
    """Add the provisioning network to a node.

    :param task: A TaskManager instance.
    :raises: NetworkError when failed to set binding:host_id
    """
    self._bind_flat_ports(task)
```

如果是neutron，会调用/usr/lib/python2.7/site-packages/ironic/drivers/modules/network/neutron.py的实现

```python
    def add_provisioning_network(self, task):
        """Add the provisioning network to a node.

        :param task: A TaskManager instance.
        :raises: NetworkError
        """
        LOG.info(_LI('Adding provisioning network to node %s'),
                 task.node.uuid)
        vifs = neutron.add_ports_to_network(
            task, CONF.neutron.provisioning_network_uuid)
        for port in task.ports:
            if port.uuid in vifs:
                internal_info = port.internal_info
                internal_info['provisioning_vif_port_id'] = vifs[port.uuid]
                port.internal_info = internal_info
                port.save()
```

不管是flat还是neutron，都依赖了neutron的实现，所以在standalone的情况下，这个配置要改成none

也就是要把node的这个参数改成noop

```shell
openstack baremetal node set ab556e18-eacb-4947-aa7c-067cdd376b49 --network-interface noop
```

这个时候，会调用到/usr/lib/python2.7/site-packages/ironic/drivers/modules/network/noop.py里的实现，而这里面都是空实现

```python
    def add_provisioning_network(self, task):
        """Add the provisioning network to a node.

        :param task: A TaskManager instance.
        """
        pass
```

所以我理解对于standalone的情况，是没有两个阶段的provision network和tenant network的，day0的时候需要配置上联，跟dhcp server是可以通的，获取到的ip将是最终的IP。

# Networking service in ironic

我们知道，当neutron作为网络服务的时候，为baremetal node分配IP的是neutron dhcp agent。在standalone的情况下，dhcp provider需要我们自己来提供，这个配置在ironic.conf中，默认是neutron

```python
[dhcp]
dhcp_provider=none
```

