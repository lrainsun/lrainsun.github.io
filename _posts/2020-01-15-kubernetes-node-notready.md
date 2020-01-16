---
layout: post
title:  "Kubernetes node NotReady"
date:   2020-01-15 22:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Env
excerpt: hostname改变导致Kubernetes node NotReady
mathjax: true
typora-root-url: ../
---

# 问题

好久没看kubernetes，感觉又变成小白了，命令都要不会敲了

今天把自己测试环境找出来，发现运行不正常了，先是连不上，后来发现是因为vm丢失了ip，dhclient获取了一下才好，然后发现三个node都not ready

```shell
[root@rain-kubernetes-1 ~]# kubectl get nodes
NAME                STATUS     ROLES    AGE   VERSION
rain-kubernetes-1   NotReady   master   83d   v1.16.2
rain-kubernetes-2   NotReady   <none>   83d   v1.16.2
rain-kubernetes-3   NotReady   <none>   83d   v1.16.2
```

# 排查

查看docker & kubelet都在running状态

```shell
[root@rain-kubernetes-1 ~]# service docker status
Redirecting to /bin/systemctl status docker.service
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2020-01-15 09:55:49 GMT; 5min ago
     Docs: https://docs.docker.com
 Main PID: 886 (dockerd)
    Tasks: 14
   Memory: 128.5M
   CGroup: /system.slice/docker.service
           └─886 /usr/bin/dockerd -H fd://
           
[root@rain-kubernetes-1 ~]# service kubelet status
Redirecting to /bin/systemctl status kubelet.service
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /usr/lib/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since Wed 2020-01-15 09:55:43 GMT; 5min ago
     Docs: https://kubernetes.io/docs/
 Main PID: 518 (kubelet)
    Tasks: 28
   Memory: 145.8M
   CGroup: /system.slice/kubelet.service
           ├─ 518 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --cgroup-driver=cgroupfs...
           └─4576 /opt/cni/bin/calico
```

进一步查看log，发现kubelet日志里全是如下错误：

```shell
[root@rain-kubernetes-1 ~]# journalctl -f -u kubelet
-- Logs begin at Wed 2020-01-15 09:55:41 GMT. --
Jan 15 10:03:00 rain-kubernetes-1.localdomain kubelet[518]: E0115 10:03:00.977530     518 kubelet.go:2267] node "rain-kubernetes-1.localdomain" not found
Jan 15 10:03:01 rain-kubernetes-1.localdomain kubelet[518]: E0115 10:03:01.077691     518 kubelet.go:2267] node "rain-kubernetes-1.localdomain" not found
Jan 15 10:03:01 rain-kubernetes-1.localdomain kubelet[518]: E0115 10:03:01.177931     518 kubelet.go:2267] node "rain-kubernetes-1.localdomain" not found
Jan 15 10:03:01 rain-kubernetes-1.localdomain kubelet[518]: E0115 10:03:01.278172     518 kubelet.go:2267] node "rain-kubernetes-1.localdomain" not found
Jan 15 10:03:01 rain-kubernetes-1.localdomain kubelet[518]: E0115 10:03:01.378431     518 kubelet.go:2267] node "rain-kubernetes-1.localdomain" not found
```

# Fix

把hostname改回rain-kubernetes-1，restart kubelet service，node就ready了

```shell
[root@rain-kubernetes-1 ~]# kubectl get nodes
NAME                STATUS     ROLES    AGE   VERSION
rain-kubernetes-1   Ready      master   83d   v1.16.2
rain-kubernetes-2   NotReady   <none>   83d   v1.16.2
rain-kubernetes-3   NotReady   <none>   83d   v1.16.2
```

为啥rain-kubernetes-1会被改成rain-kubernetes-1.localdomain呢，那是另外一个问题

