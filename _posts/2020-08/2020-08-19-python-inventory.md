---
layout: post
title:  "python修改Inventory文件"
date:   2020-08-19 23:00:00 +0800
categories: Python
tags: Python-Language
excerpt: python修改Inventory文件
mathjax: true
typora-root-url: ../
---

# python修改Inventory文件

我们需要通过输入的hosts，来拼出inventory文件

首先有一个inventory模板，其实有一些hosts的参数

inventory template:

```yaml
[all:vars]

[control]

# The network nodes are where your l3-agent and loadbalancers will run
# This can be the same as a host in the control group
[network]

[compute:children]
compute-cell1

[cell-control:children]
cell-control-cell1

[cell1:children]
compute-cell1
cell-control-cell1

[cell1:vars]
nova_cell_name= cell1
nova_cell_compute_group = compute-cell1
nova_cell_conductor_group = cell-control-cell1
nova_cell_novncproxy_group = cell-control-cell1
nova_cell_serialproxy_group = cell-control-cell1
nova_cell_spicehtml5proxy_group = cell-control-cell1

[nova-conductor:children]
cell-control

[nova-novncproxy:children]
cell-control

[nova-spicehtml5proxy:children]
cell-control

[nova-serialproxy:children]
cell-control

[cell-control-cell1]

[compute-cell1]

[storage]

[baremetal:children]
control
network
compute
cell1


# You can explicitly specify which hosts run each project by updating the
# groups in the sections below. Common services are grouped together.
[chrony-server:children]
haproxy

[chrony:children]
control
network
compute

[haproxy:children]
network

[mariadb:children]
control

[rabbitmq:children]
control

[keystone:children]
control

[glance:children]
control

[nova:children]
control

[neutron:children]
network

[openvswitch:children]
network
compute
manila-share

[cinder:children]
control

[memcached:children]
control

[horizon:children]
control

[barbican:children]
control

[ironic:children]
control

[octavia:children]
control

[placement:children]
control

# Glance
[glance-api:children]
glance

# Nova
[nova-api:children]
nova

[nova-conductor:children]
nova

[nova-super-conductor:children]
nova

[nova-novncproxy:children]
nova

[nova-scheduler:children]
nova

[nova-spicehtml5proxy:children]
nova

[nova-compute-ironic:children]
nova

[nova-serialproxy:children]
nova

# Neutron
[neutron-server:children]
control

[neutron-dhcp-agent:children]
neutron

[neutron-l3-agent:children]
neutron

[neutron-metadata-agent:children]
neutron

[neutron-ovn-metadata-agent:children]
compute

[neutron-bgp-dragent:children]
neutron

[neutron-infoblox-ipam-agent:children]
neutron

[neutron-metering-agent:children]
neutron

[ironic-neutron-agent:children]
neutron

# Cinder
[cinder-api:children]
cinder

[cinder-backup:children]
storage

[cinder-scheduler:children]
cinder

[cinder-volume:children]
storage

# Barbican
[barbican-api:children]
barbican

[barbican-keystone-listener:children]
barbican

[barbican-worker:children]
barbican

# Ironic
[ironic-api:children]
ironic

[ironic-conductor:children]
ironic

[ironic-inspector:children]
ironic

[ironic-pxe:children]
ironic

[ironic-ipxe:children]
ironic

# Octavia
[octavia-api:children]
octavia

[octavia-health-manager:children]
octavia

[octavia-housekeeping:children]
octavia

[octavia-worker:children]
octavia

# Placement
[placement-api:children]
placement

[ovn-controller:children]
ovn-controller-compute
ovn-controller-network

[ovn-controller-compute:children]
compute

[ovn-controller-network:children]
network

[ovn-database:children]
control

[ovn-northd:children]
ovn-database

[ovn-nb-db:children]
ovn-database

[ovn-sb-db:children]
ovn-database
```

hosts:

```yaml
hosts:
  - hostname: ci99qacmp001
    ipAddress: 10.100.32.36
    ssh_user: ocpadmin
    node_id: 00000001
    role: control
  - hostname: ci99qacmp002
    ipAddress: 10.100.32.37
    ssh_user: ocpadmin
    node_id: 00000002
    role: control
  - hostname: ci99qacmp003
    ipAddress: 10.100.32.38
    ssh_user: ocpadmin
    node_id: 00000003
    role: cell-control-cell1
  - hostname: ci99qacmp005
    ipAddress: 10.100.32.40
    ssh_user: ocpadmin
    node_id: 00000005
    role: cell-control-cell2
  - hostname: ci99qacmp006
    ipAddress: 10.100.32.41
    ssh_user: ocpadmin
    node_id: 00000006
    role: compute-cell1
  - hostname: ci99qacmp007
    ipAddress: 10.100.32.42
    ssh_user: ocpadmin
    node_id: 00000007
    role: compute-cell2
  - hostname: ci99qacmp008
    ipAddress: 10.100.32.43
    ssh_user: ocpadmin
    node_id: 00000008
    role: compute-cell2
```

load inventory file，生成InventoryManager对象，然后对其进行修改，我们拿到的InventoryManager就可以直接用来跑ansible，或者可以把内容回写到新的inventory file里去

```python
from ansible.inventory.manager import InventoryManager
from ansible.vars.manager import VariableManager
from ansible.parsing.dataloader import DataLoader
import yaml
from pprint import pprint

loader = DataLoader()
inventory = InventoryManager(loader, 'template')

with open('hosts.yml','r') as f:
	data = yaml.load(f)

hosts = data['hosts']

group_all = inventory.groups['all']

for host in hosts:
	group = host['role']
	if group not in inventory.groups:
		if group.startswith('compute-cell') or group.startswith('cell-control-cell'):
			type = 'compute'
			if group.startswith('compute-cell'):
				cell_id = group.lstrip('compute-cell')
			if group.startswith('cell-control-cell'):
				cell_id = group.lstrip('cell-control-cell')
				type = 'control'
			if 'cell' + cell_id not in inventory.groups:
				inventory.add_group('cell' + cell_id)

				cell_group = inventory.groups['cell' + cell_id]
				cell_group.set_variable('nova_cell_name', 'cell' + cell_id)
				cell_group.set_variable('nova_cell_compute_group', 'compute-cell' + cell_id)
				cell_group.set_variable('nova_cell_conductor_group', 'cell-control-cell' + cell_id)
				cell_group.set_variable('nova_cell_novncproxy_group', 'cell-control-cell' + cell_id)
				cell_group.set_variable('nova_cell_serialproxy_group', 'cell-control-cell' + cell_id)
				cell_group.set_variable('nova_cell_spicehtml5proxy_group', 'cell-control-cell' + cell_id)

				baremetal_group = inventory.groups['baremetal']
				baremetal_group.add_child_group(cell_group)

			cell_group = inventory.groups['cell' + cell_id]
			inventory.add_group(group)
			self_group = inventory.groups[group]
			cell_group.add_child_group(self_group)

			if cell_id != '1':
				if type == 'compute':
					compute_group = inventory.groups['compute']
					compute_group.add_child_group(self_group)
				else:
					control_group = inventory.groups['cell-control']
					control_group.add_child_group(self_group)

	inventory.add_host(host['hostname'], group)
	inventory_host = inventory.get_host(host['hostname'])
	inventory_host.set_variable('ansible_ssh_host', host['ipAddress'])
	inventory_host.set_variable('ansible_connection', 'ssh')
	inventory_host.set_variable('ansible_ssh_user', 'ocpadmin')
	inventory_host.set_variable('ansible_become', 'true')

	group_all.add_host(inventory_host)

variable = VariableManager(loader, inventory=inventory)
pprint(variable.get_vars())
```

