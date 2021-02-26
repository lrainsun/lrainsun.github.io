---
layout: post
title:  "ansible lineinfile"
date:   2020-12-27 23:00:00 +0800
categories: Ansible
tags: Ansible
excerpt: ansible lineinfile
mathjax: true
typora-root-url: ../
---

# ansible lineinfile

我们想要把公钥部署到每台host上去，但是又不想覆盖掉之前存在的public key，本来是用的copy，可以替换成lineinfile

```yaml
---
- hosts: bastion
  tasks:
    - name: mkdir .ssh
      file: name=/home/{{ user }}/.ssh state=directory owner={{ user }} group={{ user }}
      
    - name: create authorized_keys
      file: name=/home/{{ user }}/.ssh/authorized_keys state=touch owner={{ user }} group={{ user }} mode=0600
      
    - name: get public key
      local_action: shell cat {{ public_key_path }}
      register: public_key

    - name: copy public key
      lineinfile:
        path: /home/{{ user }}/.ssh/authorized_keys
        line: "{{ public_key.stdout }}"
        state: present
```

这样，如果public-key已经存在于/home/ocp/.ssh/authorized_keys就什么都不会做，如果不存在，就会把public key放进去

