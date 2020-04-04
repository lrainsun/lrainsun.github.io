---
layout: post
title:  "加载修改后的Openvswitch module"
date:   2020-03-30 23:00:00 +0800
categories: Openstack
tags: Openstack-Neutron
excerpt: 加载修改后的Openvswitch module
mathjax: true
typora-root-url: ../
---

# 加载修改后的Openvswitch module

为了要使我们编译的openvswitch kernel module生效，还需要加载到内核，具体步骤是这样的：

1. 首先生成一个空的/usr/share/openvswitch/scripts/ovs-save文件 
2. /usr/share/openvswitch/scripts/ovs-ctl force-reload-kmod，force reload openvswitch kernel module
3. ifup vlan{{ tenant_traffic_vlan }}，把traffic vlan口重新up一下
4. service neutron-openvswitch-agent restart，重启
5. 如果是network node，还需要重启l3 agent，service neutron-l3-agent restart

# 题外话

今天莫名发现一直以来几乎每天在用的工作机登录不上了，昨天还在用呢，找到cimc地址，发现被rebuild了，在没有被通知的情况下，心情很是郁闷，居然还掉眼泪了

虽然说不上来有啥重要的东西，无非是一套openstack环境+一些破脚本，可就是难过极了

《寻梦环游记》里有这样一句话，`真正的死亡，是世界上再没有一个人记得你`，虽然我还没看过这部电影，但记住了这句广告词

最宝贵的东西也不仅仅是现在拥有，回忆和习惯亦很宝贵，旧的人旧的事皆很宝贵，嗯，我是个念旧的人。

![小丸子伤心图片](/assets/images/hplgghpocv.jpeg)