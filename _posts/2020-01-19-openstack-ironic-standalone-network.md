---
galayout: post
title:  "openstack standalone ironic Networking service"
date:   2020-01-19 13:00:00 +0800
categories: Openstack
tags: Openstack-ironic
excerpt: openstack standalone ironic Networking service
mathjax: true
---

# network service in ironic

我们知道，当ironic跟openstack是集成在一起的时候，network service这块是neutron负责的，neutron会从他管理的网络里面为baremetal node分配deploy的时候使用的provision network，在deploy完成后会切换到tenant network。

这个配置选项是在ironic.conf中，默认是neutron

```python
[dhcp]

#
# From ironic
#

# DHCP provider to use. "neutron" uses Neutron, and "none"
# uses a no-op provider. (string value)
#dhcp_provider = neutron
```

/usr/lib/python2.7/site-packages/ironic/drivers/modules/network/neutron.py 实现了neutron作为dhcp provider的实现。

比如add provision network的时候，会去调用neutron的接口

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

而在standalone的情况下，这个配置要改成none

```python
[dhcp]
dhcp_provider=none
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