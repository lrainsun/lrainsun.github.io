---
layout: post
title:  "Kubernetes Service和Pod的DNS"
date:   2019-11-09 17:05:00 +0800
categories: Kubernetes
tags: Kubernetes-Service
excerpt: Kubernetes Service和Pod的DNS
mathjax: true
typora-root-url: ../
---

# Kubernetes中的DNS

`DNS` 是 Kubernetes 的核心功能之一，Kubernetes 通过 `kube-dns` 或 `CoreDNS` 作为集群的必备扩展来提供命名服务，通过 DNS 扩展，每一个 `Service` 都会产生一个独一无二的 FQDN（Fully Qualified Domain Name）名称。

在我的环境里，用的是CoreDNS：

```shell
[root@rain-kubernetes-1 rain]# kubectl get service -n kube-system
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                  AGE
kube-dns               ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP,9153/TCP   16d

[root@rain-kubernetes-1 rain]# kubectl get pods -n kube-system | grep dns
coredns-5644d7b6d9-pdfr2                    1/1     Running   0          21h
coredns-5644d7b6d9-zv6j4                    1/1     Running   0          21h
```

在Kubernetes集群上调度了DNS Pod和Service，并配置kubelet，使其告诉每个容器使用DNS Service的Ip来解析DNS名称。

# DNS名称

集群中定义的每个Service（包括DNS Service它自己）都被分配了一个DNS名称。默认的，Pod的DNS搜索列表中会包含Pod自己的命名空间和集群的默认域。

## Service

### A记录

#### Service

“正常” Service（除了 Headless Service）会以 `my-svc.my-namespace.svc.cluster-domain.example` 这种名字的形式被指派一个 DNS A 记录。 这会解析成该 Service 的 Cluster IP。

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: hostnames
spec:
  selector:
    app: hostnames
  ports:
  - name: default
    protocol: TCP
    port: 80
    targetPort: 9376
```

比如之前例子里的获取hostname的service，由于我们没有指定namespace，所以默认是建在default namespace里的

```shell
[root@rain-kubernetes-1 rain]# kubectl get service hostnames
NAME        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
hostnames   ClusterIP   10.111.231.44   <none>        80/TCP    31h
```

我们在default namespace起一个Pod，区通过域名获取该service ip，可以直接通过servicename去获取，或者全称

```shell
[root@rain-kubernetes-1 rain]# kubectl run dig --rm -it --image=docker.io/azukiapp/dig /bin/sh
/ # nslookup hostnames
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   hostnames.default.svc.cluster.local
Address: 10.111.231.44

/ # nslookup hostnames.default
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   hostnames.default.svc.cluster.local
Address: 10.111.231.44

/ # / # nslookup hostnames.default.svc.cluster.local
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   hostnames.default.svc.cluster.local
Address: 10.111.231.44
```

但如果不在一个namespace里，就不能单用servicename获取：

```shell
/ # nslookup kube-dns
Server:         10.96.0.10
Address:        10.96.0.10#53

** server can't find kube-dns: NXDOMAIN

/ # nslookup kube-dns.kube-system
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   kube-dns.kube-system.svc.cluster.local
Address: 10.96.0.10
```

#### Headless Service

有一种headless service，它跟普通service的区别在于这种service没有clusterIP（clusterIP字段的值是None）。这种service被创建后并不会被分配一个 VIP，而是会以 DNS 记录的方式暴露出它所代理的 Pod。（headless service通常被应用到有状态应用的statefulset中）

创建一个headless service，并通过dns记录访问试一下：

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: hostnames
spec:
  clusterIP: None
  selector:
    app: hostnames
  ports:
  - name: default
    protocol: TCP
    port: 80
```

```shell
/ # nslookup hostnames
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   hostnames.default.svc.cluster.local
Address: 10.168.2.21
Name:   hostnames.default.svc.cluster.local
Address: 10.168.0.10
Name:   hostnames.default.svc.cluster.local
Address: 10.168.1.12
```

“Headless” Service（没有Cluster IP）也会以 `my-svc.my-namespace.svc.cluster.local` 这种名字的形式被指派一个 DNS A 记录。 不像正常 Service，它会解析成该 Service 选择的一组 Pod 的 IP。 希望客户端能够使用这一组 IP，否则就使用标准的 round-robin 策略从这一组 IP 中进行选择。

### SRV记录

命名端口需要创建 SRV 记录，这些端口是正常 Service或 [Headless Services](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) 的一部分。 对每个命名端口，SRV 记录具有 `_my-port-name._my-port-protocol.my-svc.my-namespace.svc.cluster.local` 这种形式。 对普通 Service，这会被解析成端口号和 CNAME：`my-svc.my-namespace.svc.cluster.local`。 对 Headless Service，这会被解析成多个结果，Service 对应的每个 backend Pod 各一个， 包含 `auto-generated-name.my-svc.my-namespace.svc.cluster.local` 这种形式 Pod 的端口号和 CNAME。

```yaml
  ports:
  - name: default
    protocol: TCP
    port: 80
```

portname是"default"，protocol是"TCP"，所以可以分别试一下service和headless service的效果：

service：

```shell
/ # nslookup default.TCP.hostnames
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   default.TCP.hostnames.default.svc.cluster.local
Address: 10.104.83.227
```

headless service：

```shell
/ # nslookup default.TCP.hostnames
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   default.TCP.hostnames.default.svc.cluster.local
Address: 10.168.2.21
Name:   default.TCP.hostnames.default.svc.cluster.local
Address: 10.168.1.12
Name:   default.TCP.hostnames.default.svc.cluster.local
Address: 10.168.0.10
```

## Pods

pod会被分配一个DNS记录，名称格式为`pod-ip-address.my-namespace.pod.cluster.local`。

比如下面这个pod的dns记录的名称是10-168-0-10.default.pod.cluster.local

```shell
default       hostnames-5db5477647-8ggkg                  1/1     Running   0          3d      10.168.0.10     rain-kubernetes-1   <none>           <none>

/ # nslookup 10-168-0-10.default.pod.cluster.local
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   10-168-0-10.default.pod.cluster.local
Address: 10.168.0.10
```

# 域名解析原理

在kubernetes里启动的容器的DNS server的地址就是kubernetes里dns服务的clusterIP

```shell
[root@rain-kubernetes-1 rain]# kubectl get service -n kube-system
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                  AGE
kube-dns               ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP,9153/TCP   16d

/ # cat /etc/resolv.conf
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local qa.webex.com
options ndots:5
```

所以，所有域名的解析，其实都要经过 kubedns 的虚拟IP 10.96.0.10 进行解析，不论是 Kubernetes 内部域名还是外部的域名。

## 内部域名

当我们查找一个域名时，比如hostnames，会根据 /etc/resolve.conf 进行解析流程。选择 nameserver 10.96.0.10 进行解析，然后，用"hostnames"依次带入 /etc/resolve.conf 中的 search 域，进行DNS查找，分别是：

```shell
search default.svc.cluster.local svc.cluster.local cluster.local qa.webex.com
```

hostnames.default.svc.cluster.local -> hostnames.svc.cluster.local -> hostnames.cluster.local，直到找到为止

我们找到hostnames，或者hostnames.default，都可以完成DNS请求，但这两个不同的操作，会分别进行不同的DNS查找步骤:

```shell
//curl hostnames，一次性找到（hostnames + default.svc.cluster.local）
hostnames.default.svc.cluster.local

//curl hostnames.default，第一次找不到 (hostnames.default + default.svc.cluster.local)
hostnames.default.default.svc.cluster.local
//第二次查找可以找到 (hostnames.default + svc.cluster.local)
hostnames.default.svc.cluster.local
```

所以，在同一namespace下，通过hostnames要比通过hostnames.default查找效率更高，因为后者多了一次DNS查询

## 外部域名

那外部域名的话，如果也走上面的流程，岂不是前面几步DNS查询都会要走一遍，而显然是查不到的，/etc/resolve.conf里，还有另外一个内容：ndots

```shell
/ # cat /etc/resolv.conf
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local qa.webex.com
options ndots:5
```

ndots:5，表示：如果查询的域名包含的点“.”，不到5个，那么进行DNS查找，将使用非完全限定名称（或者叫绝对域名），如果你查询的域名包含点数大于等于5，那么DNS查询，默认会使用绝对域名进行查询。

