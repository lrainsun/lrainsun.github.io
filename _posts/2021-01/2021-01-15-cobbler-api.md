---
layout: post
title:  "cobbler add system"
date:   2021-01-15 23:01:00 +0800
categories: Linux
tags: Linux-Service
excerpt: cobbler add system
mathjax: true
typora-root-url: ../
---

# cobbler add system

我们自己维护了一个cobbler，想要call api去自动地添加system，安装os

这里有一个例子

```python
    server = 'https://' + server + '/cobbler_api'
    context = ssl.SSLContext()
    remote_server = xmlrpc.client.Server(server, context=context)
    token = remote_server.login(username, password)

    existed_host = remote_server.get_system(host)
    if existed_host and isinstance(existed_host, dict) and existed_host['name'] == host:
        print('remove existing system %s' % host)
        remote_server.remove_system(host, token)

    system_id = remote_server.new_system(token)
    remote_server.modify_system(system_id, "name", host, token)
    remote_server.modify_system(system_id, "hostname", host, token)
    remote_server.modify_system(system_id, "gateway", gateway, token)

    remote_server.modify_system(system_id, 'modify_interface', {
        "interface_type-eth0": "bond_slave",
        "interface_master-eth0": "bond0",
        "macaddress-eth0": mac,
        "interface_type-eth1": "bond_slave",
        "interface_master-eth1": "bond0",
        "interface_type-eth2": "bond_slave",
        "interface_master-eth2": "bond1",
        "interface_type-eth3": "bond_slave",
        "interface_master-eth3": "bond1",
        "interface_type-eth4": "bond_slave",
        "interface_master-eth4": "bond2",
        "interface_type-eth5": "bond_slave",
        "interface_master-eth5": "bond2",
        "interface_type-eth6": "bond_slave",
        "interface_master-eth6": "bond3",
        "interface_type-eth7": "bond_slave",
        "interface_master-eth7": "bond3",
        "ipaddress-bond0": ip,
        "bonding_opts-bond0": "mode=active-backup miimon=1000",
        "interface_type-bond0": "bond",
        "subnet-bond0": netmask,
        "gateway-bond0": gateway,
        "static-bond0": 1,
        "bonding_opts-bond1": "mode=active-backup miimon=1000",
        "interface_type-bond1": "bond",
        "static-bond1": 1,
        "bonding_opts-bond2": "mode=active-backup miimon=1000",
        "interface_type-bond2": "bond",
        "static-bond2": 1,
        "bonding_opts-bond3": "mode=active-backup miimon=1000",
        "interface_type-bond3": "bond",
        "static-bond3": 1,
    }, token)

    remote_server.modify_system(system_id, "profile", "OCP", token)
    remote_server.save_system(system_id, token)
    remote_server.sync(token)
```

一些调试

```python
print(remote_server.ping())
print(remote_server.find_profile())
print(remote_server.find_distro())
print(remote_server.find_system())
print(remote_server.get_systems())
```

