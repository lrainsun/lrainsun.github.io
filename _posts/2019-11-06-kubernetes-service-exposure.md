---
layout: post
title:  "Kubernetes Service访问"
date:   2019-11-08 17:05:00 +0800
categories: Kubernetes
tags: Kubernetes-Service
excerpt: Kubernetes从外部访问Service
mathjax: true
typora-root-url: ../
---

# 从外部访问服务

Service的IP地址，仅在集群内可达，可总会有些服务需要暴露到外部网络中接受各类客户端的访问。Service一共有四种类型：ClusterIP, NodePort, LoadBalancer, ExternalName

* ClusterIP：通过集群内部IP地址暴露服务，地址仅集群内部可达，无法被集群外部的客户端访问
* NodePort：在ClusterIP类型之上，在每个节点的IP地址的某静态端口（NodePort）暴露服务，依然会为Service分配集群IP地址，并将此作为NodePort的路由目标。外部客户端可用\<NodeIP>:\<NodePort>进行请求
* LoadBalancer：在NodePort类型之上，通过cloud provider提供的负载均衡器将服务暴露到集群外部，LoadBalancer一样具有NodePort和ClusterIP。一个LoadBalancer类型的Service会指向关联至kubernetes集群外部的，切实存在的某个负载均衡设备，该设备通过工作节点之上的NodePort向集群内发送流量
* ExternalName：如果把service发布到外部，需要配置成NodePort or LoadBalance类型，而如果要把外部的服务发布于集群内容供Pod对象使用，则需要ExternalName类型

# NodePort

把Service里之前那个获取hostname的例子改造一下

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: hostnames
spec:
  type: NodePort
  selector:
    app: hostnames
  ports:
  - nodePort: 30000
    targetPort: 9376 
    port: 80 
    protocol: TCP
    name: http
```

比上次多加了type: NodePort

```shell
[root@rain-kubernetes-1 rain]# kubectl get service
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
hostnames    NodePort    10.111.231.44   <none>        80:30000/TCP   7m28s
```

首先，上次的clusterIP访问方式依然有效

```shell
[root@rain-kubernetes-1 rain]# curl 10.111.231.44:80
hostnames-5db5477647-8ggkg
```

再用宿主机ip+nodeport试试

```shell
[root@rain-kubernetes-1 rain]# curl 10.225.28.220:30000
hostnames-5db5477647-zlg5h
```

nodePort的端口范围默认是30000-32767，如果不指定，kubernetes会随机分配一个

而nodePort的实现原理也依然是由kube-proxy生成的iptables

```shell
[root@rain-kubernetes-1 rain]# iptables-save | grep 30000
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/hostnames:http" -m tcp --dport 30000 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/hostnames:http" -m tcp --dport 30000 -j KUBE-SVC-AC2ON6IKLZPGXVMV
```

而KUBE-SVC-AC2ON6IKLZPGXVMV就是一组随机模式的iptables规则，跟上次clusterIP模式完全一样

```shell
[root@rain-kubernetes-1 rain]# iptables-save | grep 10.111.231.44
-A KUBE-SERVICES ! -s 10.168.0.0/16 -d 10.111.231.44/32 -p tcp -m comment --comment "default/hostnames:http cluster IP" -m tcp --dport 80 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.111.231.44/32 -p tcp -m comment --comment "default/hostnames:http cluster IP" -m tcp --dport 80 -j KUBE-SVC-AC2ON6IKLZPGXVMV
```

跳转到的是同一条链KUBE-SVC-AC2ON6IKLZPGXVMV

## NodePort模式下的SNAT

考虑这样一个场景：

1. Client通过node2的地址访问一个Service
2. node2上的负载均衡规则，把这个IP包转发给node1上的Pod处理
3. node1上的Pod处理完成后，会按照这个IP包的源地址发送回复
4. Client会觉得很奇怪，我明明给node2发的请求，可为啥确是node1给我回复呢？

所以可以这样：

1. Client通过node2的地址访问一个Service
2. node2上的负载均衡规则，把这个IP包转发给node1上的Pod处理
3. node2在转发之前，做一下SNAT，把原地址改成自己

这是实现是通过在POSTROUTING设置一条规则，对于标有“0x4000”的“标志的IP包，-j MASQUERADE

```shell
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE
```

> MASQUERADE是SNAT的一个特例，SNAT是指在数据包从网卡发送出去的时候，把数据包中的源地址部分替换为指定的IP；而MASQUERADE是用发送数据的网卡上的IP来替换源IP

而"0x4000"这个标志则是在DNAT之前被打上去的

```shell
-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
```

4. node1上的Pod处理完成后，会按照这个IP包的源地址发送回复
5. 这个时候原地址是node2，所以node1会把回复发给node2
6. node2把回复包转发给Client
7. Client得到了满意的答复，可是它并不知道这个包是node1回的，它还以为是node2回的呢，可是有些Client是比较在意这个事情的，非得要知道谁给我回的，那又咋办呢？

Service有个配置是externalTrafficPolicy，配成Local

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: hostnames
spec:
  type: NodePort
  externalTrafficPolicy: Local
  selector:
    app: hostnames
  ports:
  - nodePort: 30000
    targetPort: 9376
    port: 80
    protocol: TCP
    name: http
```

配成Local之后，其实宿主机上的iptables规则会被设置成只将IP包转发给本宿主机上的Pod

```shell
[root@rain-kubernetes-1 rain]# kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP            NODE                NOMINATED NODE   READINESS GATES
hostnames-5db5477647-8ggkg          1/1     Running   0          45h     10.168.0.10   rain-kubernetes-1   <none>           <none>
hostnames-5db5477647-8wfs8          1/1     Running   0          45h     10.168.1.12   rain-kubernetes-2   <none>           <none>
hostnames-5db5477647-zlg5h          1/1     Running   0          45h     10.168.2.21   rain-kubernetes-3   <none>           <none>

[root@rain-kubernetes-1 rain]# kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP            NODE                NOMINATED NODE   READINESS GATES
hostnames-5db5477647-8ggkg          1/1     Running   0          46h     10.168.0.10   rain-kubernetes-1   <none>           <none>
hostnames-5db5477647-8wfs8          1/1     Running   0          46h     10.168.1.12   rain-kubernetes-2   <none>           <none>
hostnames-5db5477647-zlg5h          1/1     Running   0          46h     10.168.2.21   rain-kubernetes-3   <none>           <none>
```

rain-kubernetes-1上，会转发到Pod hostnames-5db5477647-8ggkg

```shell
[root@rain-kubernetes-1 rain]# iptables-save | grep 30000
-A KUBE-NODEPORTS -s 127.0.0.0/8 -p tcp -m comment --comment "default/hostnames:http" -m tcp --dport 30000 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/hostnames:http" -m tcp --dport 30000 -j KUBE-XLB-AC2ON6IKLZPGXVMV

[root@rain-kubernetes-1 rain]# iptables-save | grep KUBE-XLB-AC2ON6IKLZPGXVMV
-A KUBE-XLB-AC2ON6IKLZPGXVMV -m comment --comment "Balancing rule 0 for default/hostnames:http" -j KUBE-SEP-UCPOTBP5J6CFWVH3

[root@rain-kubernetes-1 rain]# iptables-save | grep KUBE-SEP-UCPOTBP5J6CFWVH3
-A KUBE-SEP-UCPOTBP5J6CFWVH3 -p tcp -m tcp -j DNAT --to-destination 10.168.0.10:9376
```

rain-kubernetes-2上，会转发到Pod hostnames-5db5477647-8wfs8

```shell
-A KUBE-SEP-G7HB53TC4RAEG2Q6 -p tcp -m tcp -j DNAT --to-destination 10.168.1.12:9376
```

rain-kubernetes-3上，会转发到Pod hostnames-5db5477647-zlg5h

```shell
-A KUBE-SEP-3J5IS7CK5D2CCCRY -p tcp -m tcp -j DNAT --to-destination 10.168.2.21:9376
```

所以访问Service的返回结果一定是固定的：

```shell
[root@rain-kubernetes-1 rain]# curl 10.225.28.220:30000
hostnames-5db5477647-zlg5h
[root@rain-kubernetes-1 rain]# curl 10.225.28.171:30000
hostnames-5db5477647-8wfs8
[root@rain-kubernetes-1 rain]# curl 10.225.28.227:30000
hostnames-5db5477647-zlg5h
```

# LoadBalancer

在公有云提供的 Kubernetes 服务里，都使用了一个叫作 CloudProvider 的转接层，来跟公有云本身的 API 进行对接。所以，在上述 LoadBalancer 类型的 Service 被提交后，Kubernetes就会调用 CloudProvider 在公有云上为你创建一个负载均衡服务，并且把被代理的 Pod 的 IP地址配置给负载均衡服务做后端。

# ExternalName Service

使用kubernetes时有一个很常见的需求，就是当数据库部署在kubernetes集群之外的时候，集群内的service如何访问数据库呢？当然你可以直接使用数据库的IP地址和端口号来直接访问，有没有什么优雅的方式呢？你需要用到`ExternalName Service`。

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.database.example.com
  ports:
  - port: 12345
```

这个例子中，在kubernetes集群内访问`my-service`实际上会重定向到`my.database.example.com:12345`这个地址。

这时候，当你通过 Service 的 DNS 名字访问它的时候，比如访问：myservice.default.svc.cluster.local。那么，Kubernetes 为你返回的就是my.database.example.com。所以说，ExternalName 类型的 Service，其实是在 kubedns里为你添加了一条 CNAME 记录。这时，访问 my-service.default.svc.cluster.local 就和访问my.database.example.com 这个域名是一个效果了。

验证一下：

```shell
[root@rain-kubernetes-1 rain]# kubectl run dig --rm -it --image=docker.io/azukiapp/dig /bin/sh
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
If you don't see a command prompt, try pressing enter.
/ # nslookup mysql-service.default.svc.cluster.local
Server:         10.96.0.10
Address:        10.96.0.10#53

mysql-service.default.svc.cluster.local canonical name = dev-ocp2.qa.webex.com.
Name:   dev-ocp2.qa.webex.com
Address: 10.225.18.49
```

此外，Kubernetes 的 Service 还允许你为 Service 分配公有 IP 地址，比如下面这个例子：

```yam
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 9376
externalIPs:
- 80.11.12.10
```

在上述 Service 中，我为它指定的 externalIPs=80.11.12.10，那么此时，你就可以通过访问80.11.12.10:80 访问到被代理的 Pod 了。不过，在这里 Kubernetes 要求 externalIPs 必须是至少能够路由到一个 Kubernetes 的节点。

# 总结

所谓 Service，其实就是 Kubernetes 为 Pod 分配的、固定的、基于iptables（或者 IPVS）的访问入口。而这些访问入口代理的 Pod 信息，则来自于 Etcd，由kube-proxy 通过控制循环来维护。

而对iptables模式的services：

1. KUBE-SERVICES 或者 KUBE-NODEPORTS 规则对应的 Service 的入口链，这个规则应该与 VIP 和 Service 端口一一对应；
2. KUBE-SEP-(hash) 规则对应的 DNAT 链，这些规则应该与 Endpoints 一一对应；
3. KUBE-SVC-(hash) 规则对应的负载均衡链，这些规则的数目应该与 Endpoints 数目一致；
4. 如果是 NodePort 模式的话，还有 POSTROUTING 处的 SNAT 链。

