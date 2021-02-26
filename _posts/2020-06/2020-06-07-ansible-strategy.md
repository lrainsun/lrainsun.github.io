---
layout: post
title:  "Ansible strategy"
date:   2020-06-07 23:00:00 +0800
categories: Ansible
tags: Ansible-pythonapi
excerpt: Ansible strategy
mathjax: true
typora-root-url: ../
---

# Ansible strategy

ansible strategy是用来控制play的flow的，默认的strategy plugin是linear plugin：

* task的执行会被一个host batch（batch由serial定义，默认是all）中的所有host lock，每个host的单个task执行完成会等待其他（同一个host batch中的host）都完成后再执行下个任务
* 同时可以有多台host（受fork的限制）同时执行task
* 一个serial执行完成后再执行下一个serial
* 所有serial中的host执行完成后才会继续下一个task

可以修改strategy为free：

* 允许每个host batch尽快地执行到play的结尾
* ansible不会等待其他host完成当前task

# api

在python api中可以这样定义strategy

```python
        play_source = dict(
            name="Ansible Play",
            hosts=hosts,
            gather_facts=gather_facts,
            roles=roles,
            strategy=strategy,
        )
  
  play = Play().load(play_source, variable_manager=self.variable_manager, loader=self.loader)
```

# Preferences

[1] [https://docs.ansible.com/ansible/latest/plugins/strategy.html](https://docs.ansible.com/ansible/latest/plugins/strategy.html)

[2] [https://docs.ansible.com/ansible/latest/user_guide/playbooks_strategies.html](https://docs.ansible.com/ansible/latest/user_guide/playbooks_strategies.html)