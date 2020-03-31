---
layout: post
title:  "Nova中的instance_name从哪里来"
date:   2020-03-31 23:00:00 +0800
categories: Openstack
tags: Openstack-Nova
excerpt: Nova中的instance_name从哪里来
mathjax: true
typora-root-url: ../
---

# Instance_name

```shell
bash-3.2$ nova show 841ddb23-e31e-408a-9abf-876aeedcee99
+--------------------------------------+------------------------------------------------------------------------+
| Property                             | Value                                                                  |
+--------------------------------------+------------------------------------------------------------------------+
| OS-DCF:diskConfig                    | AUTO                                                                   |
| OS-EXT-AZ:availability_zone          | ta-ssd-az1                                                             |
| OS-EXT-SRV-ATTR:host                 | ci22tacmp185.webex.com                                                 |
| OS-EXT-SRV-ATTR:hostname             | rancher-server-ta-nginx                                                |
| OS-EXT-SRV-ATTR:hypervisor_hostname  | ci22tacmp185.webex.com                                                 |
| OS-EXT-SRV-ATTR:instance_name        | instance-0001f0a8                                                            
```

Qunyi说他找遍了所有的数据库表，竟没有发现这个`OS-EXT-SRV-ATTR:instance_name  `是存在哪里的，也就是数据库没有，难道是实时从libvirt拿的吗？

我找了下代码，这个instance_name原来是算出来的

 ```python
    cfg.StrOpt('instance_name_template',
               default='instance-%08x',
               help="""）
               
    base_name = CONF.instance_name_template % self.id
    
    id = Column(Integer, primary_key=True, autoincrement=True)
 ```

这边有个计算instance_name的模板，这个id就是instance数据表中的uuid

```mysql
MariaDB [nova]> select id, uuid, hostname from instances where uuid="841ddb23-e31e-408a-9abf-876aeedcee99";
+--------+--------------------------------------+-------------------------+
| id     | uuid                                 | hostname                |
+--------+--------------------------------------+-------------------------+
| 127144 | 841ddb23-e31e-408a-9abf-876aeedcee99 | rancher-server-ta-nginx |
+--------+--------------------------------------+-------------------------+
```

```python
>>> print("%08x" % 127144)
0001f0a8
```

