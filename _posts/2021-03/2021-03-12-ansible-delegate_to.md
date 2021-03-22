---
layout: post
title:  "Ansible delegate_to"
date:   2021-03-12 23:00:00 +0800
categories: Ansible
tags: Ansible
excerpt: Ansible delegate_to
mathjax: true
typora-root-url: ../
---

# delegate_to

delegate_to是ansible 的任务委派功能

默认ansible的所有task是在我们的配置的管理机器上面运行的,当在一个独立的群集里面配置,那是适用的.而有一些情况是,某些任务运行的状态是需要传递给其他机器的,在同一个任务你需要在其他机器上执行,这时候你就许多要用task委托

使用delegate_to关键字便可以配置任务在其他机器上执行.其他模块还是在所有配置的管理机器上运行的,当到了这个关键字的任务就是使用委托的机器上运行.而facts还是适用于当前的host

一个简单的应用是，比如我Limit在跑compute node，但是我还需要一些controller的fact，可以这样

```yaml
- name: Gather facts for all hosts (if using --limit)
  hosts: compute
  gather_facts: false
  vars:
    controller: '{{ groups['controller'] }}'
  tasks:
    - name: Gather facts
      setup: ''
      delegate_facts: True
      delegate_to: "{{ item }}"
      with_items: "{{ controller }}"
      when: not hostvars[item].module_setup | default(false)
```

python:

```python
    gather_control = {
        'name': 'Gather facts for control node',
        'hosts': 'all',
        'gather_facts': False,
        'vars': {
            'controller_node': '{{ groups["control"] }}'
        },
        'tasks': [{
            'name': 'Gather facts',
            'setup': '',
            'delegate_facts': True,
            'delegate_to': '{{ item }}',
            'with_items': '{{ controller_node }}',
            'when': 'not hostvars[item].module_setup | default(false)'
        }]
    }
```

