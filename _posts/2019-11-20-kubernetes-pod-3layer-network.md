---
layout: post
title:  "Kubernetes三层网络方案"
date:   2019-11-20 22:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Network
excerpt: Kubernetes三层网络方案
mathjax: true
typora-root-url: ../
---

# 三层网络方案

前面学习的是网桥类型的flannel，还有一种纯三层的网络方案，因为vxlan需要额外的封包解包，对比起来纯三层的网络方案性能更好一些。flannel还有一种host-gw模式，以及Calico项目是纯三层的方案。

Flannel支持多种Backend协议，但是不支持运行时修改Backend。官方推荐使用以下Backend：

- VXLAN，性能损耗大概在20~30%；
- host-gw, 性能损耗大概10%，要求Host之间二层直连，因此只适用于小集群；

# Flannel的host-gw模式

需要修改一下flannel配置文件：

```yaml
  net-conf.json: |
    {
      "Network": "10.168.0.0/16",
      "Backend": {
        "Type": "host-gw"
      }
    }
```

由于不支持运行时需改，所以需要把之前的flannel插件delete掉，重新创建

![image-20191120160318050](/assets/images/image-20191120160318050.png)

盗图依然~~

在我的环境里：

* rain-kubernetes-1的cni0：10.168.0.1/24
* rain-kubernetes-2的cni0：10.168.1.1/24
* rain-kubernetes-3的cni0：10.168.2.1/24

 Flannel host-gw模式启用之后，在每台host上，flannel会创建一些规则，比如rain-kubernetes-1上，到10.168.1.1/24需要经过rain-kubernetes-1的eth0，下一跳是10.225.28.171（rain-kubernetes-2的eth0的IP）

```shell
[root@rain-kubernetes-1 rain]# ip route | grep 10.168
10.168.0.0/24 dev cni0 proto kernel scope link src 10.168.0.1
10.168.1.0/24 via 10.225.28.171 dev eth0
10.168.2.0/24 via 10.225.28.227 dev eth0

[root@rain-kubernetes-2 ocp]# ip route | grep 10.168.1
10.168.1.0/24 dev cni0 proto kernel scope link src 10.168.1.1
```

所谓下一跳地址就是：如果 IP 包从主机 A 发到主机 B，需要经过路由设备 X 的中转。那么 X的 IP 地址就应该配置为主机 A 的下一跳地址。

比如rain-kubernetes-1到rain-kubernetes-2上的pod（比如10.168.1.197）

1. 下一跳是10.225.28.171（rain-kubernetes-2的eth0的IP），相当于rain-kubernetes-2充当了gateway的角色。
2. 包到了rain-kubernetes-2之后，根据route表会到cni0网桥到container里

> Flannel 子网和主机的信息，都是保存在 Etcd 当中的。flanneld 只需要 WACTH 这些数据的变化，然后实时更新路由表即可。CNI 网络插件都是可以直接连接 Kubernetes 的 APIServer 来访问 Etcd 的，无需额外部署 Etcd 给它们使用。

# Calico

Calico跟flannel的host-gw模式很像，会在每台宿主机上添加路由规则

```
< 目的容器 IP 地址段 > via < 网关的 IP 地址 > dev eth0
```

不同于 Flannel 通过 Etcd 和宿主机上的 flanneld 来维护路由信息的做法，Calico 项目使用了BGP来自动地在整个集群中分发路由信息。
暂不深入看了。。