---
layout: post
title:  "ansible run using tags"
date:   2020-08-21 23:01:00 +0800
categories: Airflow
tags: Airflow-Tutorial
excerpt: ansible run using tags
mathjax: true
typora-root-url: ../
---

# ansible run using tags

如果你有一个大型的 playbook,那能够只运行其中特定部分的配置而无需运行整个 playbook 将会很有用.

plays 和 tasks 都因这个理由而支持 “tags:”

例:

```yaml
tasks:

    - yum: name={{ item }} state=installed
      with_items:
         - httpd
         - memcached
      tags:
         - packages

    - template: src=templates/src.j2 dest=/etc/foo.conf
      tags:
         - configuration
```

如果你只想运行一个非常大的 playbook 中的 “configuration” 和 “packages”,你可以这样做:

```shell
ansible-playbook example.yml --tags "configuration,packages"
```

另一方面,如果你只想执行 playbook 中某个特定任务 *之外* 的所有任务,你可以这样做:

```shell
ansible-playbook example.yml --skip-tags "notification"
```

在context.CLIARGS可以配置

```python
context.CLIARGS = ImmutableDict(connection='smart', module_path=[self.module_path], roles_path=self.roles_path, forks=10, become=True, private_key_file='id_rsa', become_method='sudo', become_user='root', become_ask_pass=False, check=False, diff=False, verbosity=4, syntax=None, start_at_task=None, config_file='ansible.cfg', tags=['nova','neutron'], skip_tags=['glance'])
```

