---
layout: post
title:  "Ansible启动container"
date:   2021-01-28 23:00:00 +0800
categories: Ansible
tags: Ansible
excerpt: Ansible启动container
mathjax: true
typora-root-url: ../
---

# Ansible启动container

我们通过ansible启动cobbler，而由于cobbler image过大（10G+），会发生timeout

```yaml
    {
        "bastion1": {
            "task": "TASK: Start cobbler container",
            "result": "failed",
            "msg": "Error creating container: UnixHTTPConnectionPool(host='localhost', port=None): Read timed out. (read timeout=60)",
            "ignore_errors": null
        }
    }
```

我们代码是这样的

```yaml
- hosts: bastion[0]
  gather_facts: false
  tasks:
  - name: Start cobbler container
    community.general.docker_container:
      name: ocp-cobbler
      image: registry-qa.webex.com/ocp/ocp-cobbler
      network_mode: host
      privileged: yes
      tty: yes
      env:
        container: "docker"
      volumes:
        - /sys/fs/cgroup:/sys/fs/cgroup:ro
        - /run
    async: 1800
    poll: 60
```

怀疑是哪里参数设置小了timeout，然后查了https://docs.ansible.com/ansible/latest/collections/community/general/docker_container_module.html，发现

| **timeout** integer | **Default:** 60 | The maximum amount of time in seconds to wait on a response from the API.If the value is not specified in the task, the value of environment variable `DOCKER_TIMEOUT` will be used instead. If the environment variable is not set, the default value will be used. |
| ------------------- | --------------- | ------------------------------------------------------------ |
|                     |                 |                                                              |

调整了这个参数之后，发现就没有timeout的情况了

```yaml
- hosts: bastion[0]
  gather_facts: false
  tasks:
  - name: Start cobbler container
    community.general.docker_container:
      name: ocp-cobbler
      image: registry-qa.webex.com/ocp/ocp-cobbler
      network_mode: host
      privileged: yes
      tty: yes
      timeout: 300
      env:
        container: "docker"
      volumes:
        - /sys/fs/cgroup:/sys/fs/cgroup:ro
        - /run
    async: 1800
    poll: 60
```

