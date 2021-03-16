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

