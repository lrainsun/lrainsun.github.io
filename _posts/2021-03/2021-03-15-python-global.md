---
layout: post
title:  "python全局变量"
date:   2021-03-15 23:01:00 +0800
categories: Python
tags: Python-Language
excerpt: python全局变量
mathjax: true
typora-root-url: ../
---

# python全局变量

在python中，变量不需要先声明，直接使用即可，那我们怎么知道用的是局部变量还是全局变量呢？

首先：python使用的变量，在默认情况下一定是用局部变量。

其次：python如果想使用作用域之外的全局变量，则需要加global前缀。

假如说有这样一个变量

```python
PIPELINE_PARAMETERS = {'release_name': 'v4.0.80', 'security_patch_release_name': 'centos-8-stream', 'internal_host_domain': 'cloud.prv.webex.com', 'external_host_domain': 'prv.webex.com', 'internal_network_static_pool': '', 'internal_network_gwip': '192.168.0.1', 'internal_network_cidr': '192.168.0.0/20', 'storage_network_gwip': '192.168.64.1', 'storage_network_cidr': '192.168.64.0/20', 'tunnel_network_gwip': '192.168.32.1', 'tunnel_network_cidr': '192.168.32.0/20', 'data_center': 'SJC02', 'external_pxe_server': 'sjpxe.webex.com', 'external_network_vlan': '2821', 'glance_image_nfs': '10.252.67.151:/ocp_dev_sjc02_a_images', 'external_network_cidr': '10.121.245.0/24', 'external_network_gwip': '10.121.245.1', 'internal_network_vlan': '1521', 'storage_network_vlan': '1523', 'tunnel_network_vlan': '1522', 'clp_logstash_kafka_servers': 'bt1-kafka-s.webex.com:9092', 'clp_logstash_kafka_topic_id': 'bt1_logstash_infra_webex_clp', 'deployment': 'ocp-dev-sjc02-d', 'fabric_a_zoning_vsan': '11', 'fabric_b_zoning_vsan': '22', 'fc_fabric_a_address': '33', 'fc_fabric_b_address': '44', 'fc_fabric_user': '66', 'pure_san_ip': '88'}
```

我们可以这样定义全局变量

```python
for parameter in globals()['PIPELINE_PARAMETERS']:
    globals()[parameter.upper()] = globals()['PIPELINE_PARAMETERS'][parameter]
```

