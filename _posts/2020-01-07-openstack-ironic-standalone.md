---
layout: post
title:  "openstack standalone ironic环境搭建"
date:   2020-01-07 23:00:00 +0800
categories: Openstack
tags: Openstack-ironic
excerpt: standalone ironic环境搭建
mathjax: true
---

# BMAAS

The Bare Metal service manages hardware through both common (eg. PXE and IPMI) and vendor-specific remote management protocols. It provides the cloud operator with a unified interface to a heterogeneous fleet of servers while also providing the Compute service with an interface that allows physical servers to be managed as though they were virtual machines.

MAAS is an sophisticated, open-source technology that offers you the ability to discover, commission, deploy, and dynamically reconfigure a large collection of widely-varying, bare-metal servers. In short, MAAS can convert your diverse hardware investment into a cohesive, flexible, distributed data center with a minimum of time and effort.

Ironic is an OpenStack project which provisions bare metal (as opposed to virtual) machines. It may be used independently or as part of an OpenStack Cloud, and integrates with the OpenStack Identity (keystone), Compute (nova), Network (neutron), Image (glance), and Object (swift) services.

And It is possible to use the Bare Metal service without other OpenStack services. 

我们就尝试安装一下standalone的ironic服务

# 准备

尝试用kolla来做部署，需要提前安装ansible, docker，启动docker服务等。

然后下载kolla & kolla-ansible

```shell
git clone https://github.com/openstack/kolla
git clone https://github.com/openstack/kolla-ansible

pip install -r kolla/requirements.txt
pip install -r kolla-ansible/requirements.txt

pip install --ignore-installed requests
sudo pip install --ignore-installed PyYAML

cd kolla-ansible && python setup.py install
```

将globals.yml和passwords.yml复制到/etc/kolla目录

```shell
mkdir -p /etc/kolla
cp -r kolla-ansible/etc/kolla/* /etc/kolla
```

修改passwords.yml里的password，偷懒没有用kolla-genpwd，只改了我们需要的，所以不能跑precheck，会提示很多密码为空

修改/etc/kolla/globals.yml的配置，因为我们打算部署standalone的ironic，所以只安装maridb, rabbitmq, ironic

```yaml
om_rpc_transport: "rabbit"
openstack_logging_debug: "True"
enable_openstack_core: "no"
enable_mariadb: "yes"
enable_memcached: "no"
enable_rabbitmq: "{{ 'yes' if om_rpc_transport == 'rabbit' or om_notify_transport == 'rabbit' else 'no' }}"
enable_ironic: "yes"
enable_ironic_ipxe: "yes"
```

然后build deploy image，并拷贝到/etc/kolla/config/ironic下:

```
/etc/kolla/config/ironic/ironic-agent.initramfs
/etc/kolla/config/ironic/ironic-agent.kernel
```

然后跑deploy

```shell
kolla-ansible -i all-in-one deploy
```

部署完了之后，看下，ironic-api, ironic-conductor都起来了，pxe没起来，后面再看下

```shell
CONTAINER ID        IMAGE                                         COMMAND                  CREATED             STATUS                           PORTS               NAMES
b88fa650885d        kolla/centos-binary-dnsmasq:master            "dumb-init --single-…"   6 hours ago         Restarting (1) 35 seconds ago                        ironic_dnsmasq
09fa6c0c8415        kolla/centos-binary-ironic-pxe:master         "dumb-init --single-…"   6 hours ago         Up 6 hours                                           ironic_ipxe
a9092c178df3        kolla/centos-binary-ironic-pxe:master         "dumb-init --single-…"   6 hours ago         Restarting (71) 35 seconds ago                       ironic_pxe
7d71a41b56ad        kolla/centos-binary-ironic-inspector:master   "dumb-init --single-…"   6 hours ago         Restarting (0) 29 seconds ago                        ironic_inspector
57ae8af490a3        kolla/centos-binary-ironic-api:master         "dumb-init --single-…"   6 hours ago         Up 6 hours                                           ironic_api
2342db063632        kolla/centos-binary-ironic-conductor:master   "dumb-init --single-…"   6 hours ago         Up 6 hours                                           ironic_conductor
f2ebe0cceb16        kolla/centos-binary-rabbitmq:master           "dumb-init --single-…"   8 hours ago         Up 6 hours                                           rabbitmq
6efc3ec66cd7        kolla/centos-binary-iscsid:master             "dumb-init --single-…"   8 hours ago         Up 6 hours                                           iscsid
2e876114b2b0        kolla/centos-binary-mariadb:master            "dumb-init -- kolla_…"   8 hours ago         Up 6 hours                                           mariadb
```

然后看起来curl是work的

```shell
[root@rain-ironic-standalone ~]# curl 10.121.246.183:6385/v1/ | jq .
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1041  100  1041    0     0   185k      0 --:--:-- --:--:-- --:--:--  203k
{
  "media_types": [
    {
      "base": "application/json",
      "type": "application/vnd.openstack.ironic.v1+json"
    }
  ],
  "version": {
    "status": "CURRENT",
    "min_version": "1.1",
    "version": "1.59",
    "id": "v1",
    "links": [
      {
        "href": "http://10.121.246.183:6385/v1/",
        "rel": "self"
      }
    ]
  },
  "links": [
    {
      "href": "http://10.121.246.183:6385/v1/",
      "rel": "self"
    },
    {
      "href": "https://docs.openstack.org//ironic/latest/contributor//webapi.html",
      "type": "text/html",
      "rel": "describedby"
    }
  ],
  "drivers": [
    {
      "href": "http://10.121.246.183:6385/v1/drivers/",
      "rel": "self"
    },
    {
      "href": "http://10.121.246.183:6385/drivers/",
      "rel": "bookmark"
    }
  ],
  "ports": [
    {
      "href": "http://10.121.246.183:6385/v1/ports/",
      "rel": "self"
    },
    {
      "href": "http://10.121.246.183:6385/ports/",
      "rel": "bookmark"
    }
  ],
  "chassis": [
    {
      "href": "http://10.121.246.183:6385/v1/chassis/",
      "rel": "self"
    },
    {
      "href": "http://10.121.246.183:6385/chassis/",
      "rel": "bookmark"
    }
  ],
  "nodes": [
    {
      "href": "http://10.121.246.183:6385/v1/nodes/",
      "rel": "self"
    },
    {
      "href": "http://10.121.246.183:6385/nodes/",
      "rel": "bookmark"
    }
  ],
  "id": "v1"
}
```

可不知道为什么cli总提示出错，待解决

```shell
[root@rain-ironic-standalone ~]# openstack baremetal node list
Unexpected exception for http://10.121.246.183:6385/v1/nodes: Failed to parse: http://10.121.246.183:6385/v1/nodes
```

