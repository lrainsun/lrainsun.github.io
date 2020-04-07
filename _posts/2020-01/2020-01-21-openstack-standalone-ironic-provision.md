---
layout: post
title:  "openstack standalone ironic部署baremetal node"
date:   2020-01-21 17:00:00 +0800
categories: Openstack
tags: Openstack-Ironic
excerpt: openstack standalone ironic部署baremetal node
mathjax: true
---

# DHCP配置

ironic_dnsmasq充当baremetal node的dhcpserver，配置如下

```shell
# NOTE(yoctozepto): ironic-dnsmasq is used to deliver DHCP(v6) service
# DNS service is disabled:
port=0

interface=eth0
bind-interfaces

dhcp-range=10.225.22.6,10.225.22.14,255.255.255.0
dhcp-sequential-ip

dhcp-option=option:tftp-server,10.225.17.62
dhcp-option=option:server-ip-address,10.225.17.62
dhcp-option=210,/tftpboot/
dhcp-option=3,10.225.22.1
dhcp-option=option:bootfile-name,pxelinux.0
```

# Provision

在deploy baremetal node之前，首先需要注册一个node，我们使用impi driver来验证流程

```shell
openstack baremetal node create \
	--driver ipmi --name 10.225.11.111 \
	--driver-info ipmi_address=10.225.11.111 \
	--driver-info ipmi_username=admin \
	--driver-info ipmi_password=****** \
	--driver-info deploy_kernel=file:///tftpboot/ironic-agent.kernel \
	--driver-info deploy_ramdisk=file:///tftpboot/ironic-agent.initramfs
	
openstack baremetal node manage ab556e18-eacb-4947-aa7c-067cdd376b49
openstack baremetal node provide ab556e18-eacb-4947-aa7c-067cdd376b49

[root@rain-standalone-ironic ~]# openstack baremetal node show ab556e18-eacb-4947-aa7c-067cdd376b49 | grep interface
| bios_interface         | no-bios                                                                                                                                                                                                                                                                                                              |
| boot_interface         | pxe                                                                                                                                                                                                                                                                                                                  |
| console_interface      | no-console                                                                                                                                                                                                                                                                                                           |
| deploy_interface       | iscsi                                                                                                                                                                                                                                                                                                                |
| inspect_interface      | no-inspect                                                                                                                                                                                                                                                                                                           |
| management_interface   | ipmitool                                                                                                                                                                                                                                                                                                             |
| network_interface      | noop                                                                                                                                                                                                                                                                                                                 |
| power_interface        | ipmitool                                                                                                                                                                                                                                                                                                             |
| raid_interface         | no-raid                                                                                                                                                                                                                                                                                                              |
| rescue_interface       | no-rescue                                                                                                                                                                                                                                                                                                            |
| storage_interface      | noop                                                                                                                                                                                                                                                                                                                 |
| vendor_interface       | ipmitool
```

然后把network inteface改成noop

```shell
openstack baremetal node set ab556e18-eacb-4947-aa7c-067cdd376b49 --network-interface noop
```

创建一个port

```shell
openstack baremetal port create --pxe-enabled True --node ab556e18-eacb-4947-aa7c-067cdd376b49 fc:99:47:49:27:50
```

# Deploy

对node设置instance info

```shell
NODE_UUID=ab556e18-eacb-4947-aa7c-067cdd376b49
IMG=http://10.225.12.134/ubuntu-16.04-server-cloudimg-amd64-disk1.img
MD5HASH=81ba5e5d97556921db8fc815420d5848
openstack baremetal node set $NODE_UUID \
    --instance-info image_source=$IMG \
    --instance-info image_checksum=$MD5HASH \
    --instance-info root_gb=10 \
    --instance-info capabilities="{\"boot_option\": \"local\"}"
```

这是对一个baremetal node的实例的赋值，一旦node被undeploy就会被清掉。下面开始deploy：

```shell
openstack baremetal node deploy ab556e18-eacb-4947-aa7c-067cdd376b49
```

deploy完成之后

```shell
[root@rain-standalone-ironic ~]# openstack baremetal node list
+--------------------------------------+------+---------------+-------------+--------------------+-------------+
| UUID                                 | Name | Instance UUID | Power State | Provisioning State | Maintenance |
+--------------------------------------+------+---------------+-------------+--------------------+-------------+
| ab556e18-eacb-4947-aa7c-067cdd376b49 | None | None          | power on    | active             | False       |
+--------------------------------------+------+---------------+-------------+--------------------+-------------+
```

