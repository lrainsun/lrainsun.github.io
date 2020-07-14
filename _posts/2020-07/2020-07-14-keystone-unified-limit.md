---
layout: post
title:  "openstack代码路径"
date:   2020-07-14 23:00:00 +0800
categories: Openstack
tags: Openstack-Keystone
excerpt: openstack代码路径
mathjax: true
typora-root-url: ../
---

# openstack代码路径

我们搭建了新的u版本的openstack container环境，新版只支持python3.6了，代码路径有变更，记录一下，好找

比如nova，keystone

/var/lib/kolla/venv/lib/python3.6/site-packages/nova/

/var/lib/kolla/venv/lib/python3.6/site-packages/keystone/

horizon在

/var/lib/kolla/venv/lib/python3.6/site-packages/openstack_dashboard/

/var/lib/kolla/venv/lib/python3.6/site-packages/openstack_auth

可以看到现在代码是在virtualenv下了

