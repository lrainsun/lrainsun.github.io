---
layout: post
title:  "从远程主机上下载文件"
date:   2020-06-11 23:00:00 +0800
categories: Ansible
tags: Ansible-pythonapi
excerpt: 从远程主机上下载文件
mathjax: true
typora-root-url: ../
---

# 从远程主机上下载文件

现在在airflow上跑ansible，而ssh用到的public和private key都在service node上：

* 当新增加node的时候，我们需要把public key（也在service node上）拷贝到各个node上
* 还要把service node上的private key下载到airflow上，这样就可以用这个private key去跑ansible了

一开始想着用ansible来跑，ansible有个copy的module，是从本地拷贝文件到远程，还有一个module叫fetch，是反过来的，从远程拷贝文件到本地

试了一下

```yaml
- hosts: monitor
  tasks:
    - name: copy public key from service node
      fetch: src=/home/ocpadmin/.ssh/id_rsa.pub dest=/sshkey/
```

这样会把service node上的public key拷贝到本地，但并不是直接拷贝到/sshkey/目录下

假如我的service node hostname是ci81hf1cmp004，就会被拷贝到`/sshkey/ci81hf1cmp004/home/ocpadmin/.ssh/id_rsa.pub`，这个跟我的预期是不一样的

后来尝试用python，可以用scp直接下载到本地

```python
import paramiko
from scp import SCPClient
client = paramiko.SSHClient()
client.load_system_host_keys()
client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
client.connect('ci81hf1cmp004.qa.webex.com', port=22, username='ocpadmin', password='******')
scp = SCPClient(client.get_transport())
scp.get('~/.ssh/id_rsa.pub', '/Users/minsu/id_rsa.pub')
```



