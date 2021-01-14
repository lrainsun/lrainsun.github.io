---
layout: post
title:  "Ansible vars"
date:   2021-01-15 23:00:00 +0800
categories: Airflow
tags: Airflow-Tutorial
excerpt: Ansible vars
mathjax: true
typora-root-url: ../
---

# Ansible vars

openstack新版本我们用了Kolla-ansible作部署，但是会有一些自己的修改，为了让Kolla-ansible跟社区保持同步，我们的修改都提取到另外的文件夹里，然后通过覆盖或者link的方式

比如，我们想要替换一个template，之前用的是文件覆盖的方式

```yaml
  - name: Customize haproxy external
    local_action: copy src={{ item.src }} dest={{ item.dest }}
    with_items:
      - { src: "default_parameters/prometheus_default", dest: "{{ playbook_dir }}/../../kolla-ansible/ansible/roles/prometheus/defaults/main.yml" }
      - { src: "default_parameters/rabbitmq_default", dest: "{{ playbook_dir }}/../../kolla-ansible/ansible/roles/rabbitmq/defaults/main.yml" }
    run_once: true

```

用我们自己的模板prometheus_default去替换原生的roles/prometheus/defaults/main.yml

然而在跑起来的时候，发现虽然文件被替换了，但是实际deploy并没有生效，用的还是原先的配置

这个是因为prometheus_default里是这样的

```yaml
---
project_name: "prometheus"

prometheus_services:
  prometheus-server:
    container_name: prometheus_server
    group: prometheus

```

定义的是一些vars，而ansible会在一开始就把这些vars load出来，在这之后再去修改文件，并不会重新load vars

所以修改成了下面这样

```yaml
  - name: set fact for prometheus variables
    include_vars: default_parameters/prometheus_default

  - name: set fact for rabbitmq variables
    include_vars: default_parameters/rabbitmq_default
```

同样的问题在handler定义里面也会有，所以handler目前还没有想到好的办法替换，只能是重新写一份

