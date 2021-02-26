---
layout: post
title:  "ansible jump server"
date:   2020-09-01 23:00:00 +0800
categories: Ansible
tags: Ansible-pythonapi
excerpt: ansible jump server
mathjax: true
typora-root-url: ../
---

# ansible jump server

在我们新的ocp Deployment部署里，management网络会变成private network，对外只暴露api的网络。这就意味着我们没办法直接从外部访问到所有的host（controller node, compute node等），只能通过跳板机，而我们现在部署是通过airflow来跑ansible，也没有固定的ansible server。ansible可以支持通过跳板机来访问target server

## How do I configure a jump host to access servers that I have no direct access to?

You can set a ProxyCommand in the ansible_ssh_common_args inventory variable. Any arguments specified in this variable are added to the sftp/scp/ssh command line when connecting to the relevant host(s). Consider the following inventory group:

```
[gatewayed]
foo ansible_host=192.0.2.1
bar ansible_host=192.0.2.2
```

You can create group_vars/gatewayed.yml with the following contents:

```
ansible_ssh_common_args: '-o ProxyCommand="ssh -W %h:%p -q user@gateway.example.com"'
```

Ansible will append these arguments to the command line when trying to connect to any hosts in the group gatewayed. (These arguments are used in addition to any ssh_args from ansible.cfg, so you do not need to repeat global ControlPersist settings in ansible_ssh_common_args.)

Note that ssh -W is available only with OpenSSH 5.4 or later. With older versions, it’s necessary to execute nc %h:%p or some equivalent command on the bastion host.

With earlier versions of Ansible, it was necessary to configure a suitable ProxyCommand for one or more hosts in ~/.ssh/config, or globally by setting ssh_args in ansible.cfg

我们可以通过把这个配置到配置文件里，但是这样就不灵活了，对于不同的环境我们的jump server应该是不一致的。

所以通过ansible_ssh_common_args来传递应该是最好的

```shell
ansible-playbook -i private-inventory check-release.yml --private-key key --ssh-common-args='-o ProxyCommand="ssh -i key -o StrictHostKeyChecking=no -o ControlMaster=auto -W %h:%p -q ocp@10.225.17.252"'
```

`-o StrictHostKeyChecking=no -o ControlMaster=auto`如果不加的话，会要求在登录每台host的时候手工输入yes，我们强制要求不去check hostkey

可以看到在ansible-playbook的时候我们传入了`--private-key key`，而在ProxyCommand里我们又传入了一个key`-i key`

前者的private-key是给我们的target host用的，也就是我们要在哪些host run playbook

后者的key是给我们jump server用的，通过哪个key来登录jump server

这两个key都需要，缺一不可

