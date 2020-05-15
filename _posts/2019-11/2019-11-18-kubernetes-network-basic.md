---
layout: post
title:  "Kubernetes网络模型基础"
date:   2019-11-18 14:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Network
excerpt: Kubernetes网络模型基础
mathjax: true
typora-root-url: ../
---

# kubernetes网络模型基础

kubernetes网络中主要存在四种类型的通信：

1. 同一Pod内的容器间通信
2. Pod之间的通信
3. Pod与Service间的通信
4. cluster外部流量同Service间的通信

kubernetes为Pod和Service资源对象分别使用了各自的专用网络：

* Pod网络由kubernetes网络插件配置实现（配置网络插件的时候需要指定）

```shell
[root@rain-kubernetes-1 rain]# kubectl get pod -o wide | grep rain-kubernetes-1
coffee-8c8ff9b4f-ftnfh            1/1     Running     0          6d15h   10.168.0.12    rain-kubernetes-1   <none>           <none>

[root@rain-kubernetes-1 rain]# kubectl exec -it coffee-8c8ff9b4f-ftnfh -- /bin/sh
/ # ifconfig
eth0      Link encap:Ethernet  HWaddr CE:02:AE:31:B1:46
          inet addr:10.168.0.12  Bcast:0.0.0.0  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:912 errors:0 dropped:0 overruns:0 frame:0
          TX packets:33 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:41304 (40.3 KiB)  TX bytes:3760 (3.6 KiB)
```

我用的网络插件是flannel，创建的时候需要指定Pod IP

```yaml
  net-conf.json: |
    {
      "Network": "10.168.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
```

* Service网络由kubernetes集群予以指定 （集群负责配置和管理，--service-cluster-ip-range=10.96.0.0/12）

```shell
[root@rain-kubernetes-1 rain]# kubectl get pod kube-apiserver-rain-kubernetes-1 -n kube-system -o yaml | grep service-cluster-ip-range
    - --service-cluster-ip-range=10.96.0.0/12
```

kubernetes网络模型需要外部插件实现，要求满足：

* 所有Pod间均可不经NAT机制直接通信
* 所有节点均可不经NAT机制直接与所有容器通信
* 容器自己使用的IP地址也是其他容器或节点直接看到的地址，所有Pod都位于同一平面网络中，而且可以使用Pod自身的地址直接通信

kubernetes使用的网络查件必须能够为Pod提供满足以上要求的网络，需要为每个Pod配置至少一个特定的地址（Pod IP），Pod IP实际存在于某个网卡（可以是虚拟设备）上

而Service地址却是一个虚拟IP地址，没有任何网络接口配置此地址，由kube-proxy借助iptables规则或者ipvs规则重新定向到本地端口，再将其调度到后端Pod对象。Service的IP地址是集群提供服务的地址，也称为cluster ip。

# 网络模型

![image-20191118134657395](/../assets/images/image-20191118134657395.png)

kubernetes集群至少应该包含三个网络：

1. 节点网络：
   1. 各主机（Master，Node，etcd）等自身所属的网络
   2. 配置于主机的网络接口，用于各主机之间的通信
   3. 这个地址在kubernetes集群建立之间就配置好了，不是kubernetes管理的
2. Pod网络：
   1. 为各Pod对象设置IP地址等网络参数
   2. 配置于Pod中容器的网络接口上
   3. 借助于cni插件实现
3. Service网络:
   1. 是一个虚拟网络，用于为kubernetes集群中Service配置IP地址
   2. 没有任何网络接口配置此地址，由kube-proxy借助iptables规则或者ipvs规则重新定向到本地端口，再将其调度到后端Pod对象
   3. 在kubernetes集群创建时指定，Service地址在创建Service时动态配置

# 集群上的网络通信

![image-20191118140152888](/../assets/images/image-20191118140152888.png)

kubernetes集群客户端可以分为两类：

* API Server客户端
* 应用程序客户端