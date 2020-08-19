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

[cell-control:children]

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

[deployment]
[baremetal:children]
control
network
compute
storage
monitoring
cell1

[tls-backend:children]
control

# You can explicitly specify which hosts run each project by updating the
# groups in the sections below. Common services are grouped together.
[chrony-server:children]
haproxy

[chrony:children]
control
network
compute
storage
monitoring

[collectd:children]
compute

[grafana:children]
monitoring

[etcd:children]
control

[influxdb:children]
monitoring

[prometheus:children]
monitoring

[kafka:children]
control

[karbor:children]
control

[kibana:children]
control

[telegraf:children]
compute
control
monitoring
network
storage

[elasticsearch:children]
control

[haproxy:children]
network

[hyperv]
#hyperv_host

[hyperv:vars]
#ansible_user=user
#ansible_password=password
#ansible_port=5986
#ansible_connection=winrm
#ansible_winrm_server_cert_validation=ignore

[mariadb:children]
control

[rabbitmq:children]
control

[outward-rabbitmq:children]
control

[qdrouterd:children]
control

[monasca-agent:children]
compute
control
monitoring
network
storage

[monasca:children]
monitoring

[storm:children]
monitoring

[mongodb:children]
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

[cloudkitty:children]
control

[freezer:children]
control

[memcached:children]
control

[horizon:children]
control

[swift:children]
control

[barbican:children]
control

[heat:children]
control

[murano:children]
control

[solum:children]
control

[ironic:children]
control

[magnum:children]
control

[qinling:children]
control

[sahara:children]
control

[mistral:children]
control

[manila:children]
control

[ceilometer:children]
control

[aodh:children]
control

[cyborg:children]
control
compute

[congress:children]
control

[panko:children]
control

[gnocchi:children]
control

[tacker:children]
control

[trove:children]
control

# Tempest
[tempest:children]
control

[senlin:children]
control

[vmtp:children]
control

[vitrage:children]
control

[watcher:children]
control

[rally:children]
control

[searchlight:children]
control

[octavia:children]
control

[designate:children]
control

[placement:children]
control

[bifrost:children]
deployment

[zookeeper:children]
control

[zun:children]
control

[skydive:children]
monitoring

[redis:children]
control

[blazar:children]
control

# Additional control implemented here. These groups allow you to control which
# services run on which hosts at a per-service level.
#
# Word of caution: Some services are required to run on the same host to
# function appropriately. For example, neutron-metadata-agent must run on the
# same host as the l3-agent and (depending on configuration) the dhcp-agent.

# Elasticsearch Curator
[elasticsearch-curator:children]
elasticsearch

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

# Cloudkitty
[cloudkitty-api:children]
cloudkitty

[cloudkitty-processor:children]
cloudkitty

# Freezer
[freezer-api:children]
freezer

[freezer-scheduler:children]
freezer

# iSCSI
[iscsid:children]
compute
storage
ironic

[tgtd:children]
storage

# Karbor
[karbor-api:children]
karbor

[karbor-protection:children]
karbor

[karbor-operationengine:children]
karbor

# Manila
[manila-api:children]
manila

[manila-scheduler:children]
manila

[manila-share:children]
network

[manila-data:children]
manila

# Swift
[swift-proxy-server:children]
swift

[swift-account-server:children]
storage

[swift-container-server:children]
storage

[swift-object-server:children]
storage

# Barbican
[barbican-api:children]
barbican

[barbican-keystone-listener:children]
barbican

[barbican-worker:children]
barbican

# Heat
[heat-api:children]
heat

[heat-api-cfn:children]
heat

[heat-engine:children]
heat

# Murano
[murano-api:children]
murano

[murano-engine:children]
murano

# Monasca
[monasca-agent-collector:children]
monasca-agent

[monasca-agent-forwarder:children]
monasca-agent

[monasca-agent-statsd:children]
monasca-agent

[monasca-api:children]
monasca

[monasca-grafana:children]
monasca

[monasca-log-transformer:children]
monasca

[monasca-log-persister:children]
monasca

[monasca-log-metrics:children]
monasca

[monasca-thresh:children]
monasca

[monasca-notification:children]
monasca

[monasca-persister:children]
monasca

# Storm
[storm-worker:children]
storm

[storm-nimbus:children]
storm

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

# Magnum
[magnum-api:children]
magnum

[magnum-conductor:children]
magnum

# Qinling
[qinling-api:children]
qinling

[qinling-engine:children]
qinling

# Sahara
[sahara-api:children]
sahara

[sahara-engine:children]
sahara

# Solum
[solum-api:children]
solum

[solum-worker:children]
solum

[solum-deployer:children]
solum

[solum-conductor:children]
solum

[solum-application-deployment:children]
solum

[solum-image-builder:children]
solum

# Mistral
[mistral-api:children]
mistral

[mistral-executor:children]
mistral

[mistral-engine:children]
mistral

[mistral-event-engine:children]
mistral

# Ceilometer
[ceilometer-central:children]
ceilometer

[ceilometer-notification:children]
ceilometer

[ceilometer-compute:children]
compute

[ceilometer-ipmi:children]
compute

# Aodh
[aodh-api:children]
aodh

[aodh-evaluator:children]
aodh

[aodh-listener:children]
aodh

[aodh-notifier:children]
aodh

# Cyborg
[cyborg-api:children]
cyborg

[cyborg-agent:children]
compute

[cyborg-conductor:children]
cyborg

# Congress
[congress-api:children]
congress

[congress-datasource:children]
congress

[congress-policy-engine:children]
congress

# Panko
[panko-api:children]
panko

# Gnocchi
[gnocchi-api:children]
gnocchi

[gnocchi-statsd:children]
gnocchi

[gnocchi-metricd:children]
gnocchi

# Trove
[trove-api:children]
trove

[trove-conductor:children]
trove

[trove-taskmanager:children]
trove

# Multipathd
[multipathd:children]
compute
storage

# Watcher
[watcher-api:children]
watcher

[watcher-engine:children]
watcher

[watcher-applier:children]
watcher

# Senlin
[senlin-api:children]
senlin

[senlin-conductor:children]
senlin

[senlin-engine:children]
senlin

[senlin-health-manager:children]
senlin

# Searchlight
[searchlight-api:children]
searchlight

[searchlight-listener:children]
searchlight

# Octavia
[octavia-api:children]
octavia

[octavia-health-manager:children]
octavia

[octavia-housekeeping:children]
octavia

[octavia-worker:children]
octavia

# Designate
[designate-api:children]
designate

[designate-central:children]
designate

[designate-producer:children]
designate

[designate-mdns:children]
network

[designate-worker:children]
designate

[designate-sink:children]
designate

[designate-backend-bind9:children]
designate

# Placement
[placement-api:children]
placement

# Zun
[zun-api:children]
zun

[zun-wsproxy:children]
zun

[zun-compute:children]
compute

[zun-cni-daemon:children]
compute

# Skydive
[skydive-analyzer:children]
skydive

[skydive-agent:children]
compute
network

# Tacker
[tacker-server:children]
tacker

[tacker-conductor:children]
tacker

# Vitrage
[vitrage-api:children]
vitrage

[vitrage-notifier:children]
vitrage

[vitrage-graph:children]
vitrage

[vitrage-ml:children]
vitrage

[vitrage-persistor:children]
vitrage

# Blazar
[blazar-api:children]
blazar

[blazar-manager:children]
blazar

# Prometheus
[prometheus-node-exporter:children]
monitoring
control
compute
network
storage

[prometheus-mysqld-exporter:children]
mariadb

[prometheus-haproxy-exporter:children]
haproxy

[prometheus-memcached-exporter:children]
memcached

[prometheus-cadvisor:children]
monitoring
control
compute
network
storage

[prometheus-alertmanager:children]
monitoring

[prometheus-openstack-exporter:children]
monitoring

[prometheus-elasticsearch-exporter:children]
elasticsearch

[prometheus-blackbox-exporter:children]
monitoring

[masakari-api:children]
control

[masakari-engine:children]
control

[masakari-monitors:children]
compute

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
			if group.startswith('compute-cell'):
				cell_id = group.lstrip('compute-cell')
			if group.startswith('cell-control-cell'):
				cell_id = group.lstrip('cell-control-cell')
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
			compute_group = inventory.groups[group]
			cell_group.add_child_group(compute_group)

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

