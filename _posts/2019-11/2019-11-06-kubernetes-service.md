---
layout: post
title:  "Kubernetes Service"
date:   2019-11-06 18:03:00 +0800
categories: Kubernetes
tags: Kubernetes-Service
excerpt: Kubernetes中的Service
mathjax: true
typora-root-url: ../
---

# Service的概念

Kubernetes Pod是有生命周期的。他们可以被创建，而且销毁不会再启动。 所以虽然每个Pod都有自己的IP地址，但是不能保证随着时间推移，旧的Pod还存在不会被新的Pod所替代。而且为了高可用性，有可能创建多个Pod实例来完成同样一件事情。所以：

1. Pod的IP不是固定的
2. 一组Pod实例之间总会有负载均衡的需求

这就是为啥需要Service了，Service 是一组 Pod 的逻辑集合和访问方式的抽象，也可以把 Service 加上的一组 Pod 称作是一个微服务。

# 创建Service

我们先创建一个Deployment，里面包含了三个Pod，访问每个Pod的9376端口，就会返回自己的hostname

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostnames
spec:
  selector:
    matchLabels:
      app: hostnames
  replicas: 3
  template:
    metadata:
      labels:
        app: hostnames
    spec:
      containers:
      - name: hostnames
        image: k8s.gcr.io/serve_hostname
        ports:
        - containerPort: 9376
          protocol: TCP
```

```shell
[root@rain-kubernetes-1 rain]# kubectl get pods -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP            NODE                NOMINATED NODE   READINESS GATES
hostnames-5db5477647-8ggkg          1/1     Running   0          19s     10.168.0.10   rain-kubernetes-1   <none>           <none>
hostnames-5db5477647-8wfs8          1/1     Running   0          19s     10.168.1.12   rain-kubernetes-2   <none>           <none>
hostnames-5db5477647-zlg5h          1/1     Running   0          19s     10.168.2.21   rain-kubernetes-3   <none>           <none>

[root@rain-kubernetes-1 rain]# curl 10.168.0.10:9376
hostnames-5db5477647-8ggkg
[root@rain-kubernetes-1 rain]# curl 10.168.1.12:9376
hostnames-5db5477647-8wfs8
[root@rain-kubernetes-1 rain]# curl 10.168.2.21:9376
hostnames-5db5477647-zlg5h
```

给这三个Pod创建一个Service

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: hostnames
spec:
  selector:          # 声明这个Service只代理携带了app=hostnames标签的Pod
    app: hostnames
  ports:
  - name: default
    protocol: TCP
    port: 80    # Service的80端口
    targetPort: 9376  # 代理了Pod的9376端口
```

这个Service只代理携带了app=hostnames标签的Pod，那么反过来，只要Pod携带了app=hostnames的Label，就会被选做backend呢？并不是的，只有是处于Running状态，而且readinessProbe检查通过的Pod才有这个资格。当某一个Pod出现问题时，Kubernetes会自动把它从Service里摘除掉。

而且，被选中的Pod有个名字，叫Service的Endpoints

```shell
[root@rain-kubernetes-1 rain]# kubectl get endpoints hostnames
NAME        ENDPOINTS                                            AGE
hostnames   10.168.0.10:9376,10.168.1.12:9376,10.168.2.21:9376   34m
```

```shell
[root@rain-kubernetes-1 rain]# kubectl get service hostnames
NAME        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
hostnames   ClusterIP   10.98.154.209   <none>        80/TCP    37m
```

这时，通过service的VIP地址的80端口就可以访问到代理的Pod，可以看到Service提供的是Round Robin方式的负载均衡。

```shell
[root@rain-kubernetes-1 rain]# curl 10.98.154.209:80
hostnames-5db5477647-8wfs8
[root@rain-kubernetes-1 rain]# curl 10.98.154.209:80
hostnames-5db5477647-8ggkg
[root@rain-kubernetes-1 rain]# curl 10.98.154.209:80
hostnames-5db5477647-zlg5h
```

![image-20191106162226943](/assets/images/image-20191106162226943.png)

Service对象的IP地址称为Cluster IP，它位于kubernetes cluster配置指定的专用IP地址范围之内（启动api-server时--service-cluster-ip-range=10.96.0.0/12），而且是一种虚拟IP地址，在service对象创建后即保持不变，能够被同一集群中的Pod资源所访问。Service端口接收客户端请求并转发到后端的Pod中应用的相应端口上，这种代理机制也称为port proxy，或者四层代理，工作与TCP/IP协议栈的传输层。

# 工作原理

Service是由kube-proxy组件，加上iptables来共同实现的。
对于我们创建的名叫hostnames的Service来说，一旦它被提交给Kubernetes，那么 kube-proxy 就可以通过 Service的Informer感知到这样一个Service对象的添加。而作为对这个事件的响应，它就会在宿主机上创建这样一条iptables规则：

```shell
-A PREROUTING -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A OUTPUT -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A INPUT -m conntrack --ctstate NEW -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A FORWARD -m conntrack --ctstate NEW -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
-A OUTPUT -m conntrack --ctstate NEW -m comment --comment "kubernetes service portals" -j KUBE-SERVICES
```

```shell
[root@rain-kubernetes-1 rain]# iptables-save | grep KUBE-SERVICES | grep 10.98.154.209
-A KUBE-SERVICES -d 10.98.154.209/32 -p tcp -m comment --comment "default/hostnames:default cluster IP" -m tcp --dport 80 -j KUBE-SVC-ODX2UBAZM7RQWOIU
```

这条 iptables 规则的含义是：凡是目的地址是10.98.154.209、目的端口是80的 IP包，都应该跳转到另外一条名叫KUBE-SVC-ODX2UBAZM7RQWOIU的iptables链进行处理。

这一条规则，就为这个Service 设置了一个固定的入口地址。并且，由于10.98.154.209只是一条iptables规则上的配置，并没有真正的网络设备，所以你ping这个地址，是不会有任何响应的。

```shell
[root@rain-kubernetes-1 rain]# iptables-save | grep KUBE-SVC-ODX2UBAZM7RQWOIU
-A KUBE-SVC-ODX2UBAZM7RQWOIU -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-UBFECYXO7WGMGUOV
-A KUBE-SVC-ODX2UBAZM7RQWOIU -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-SGM2NWFJJ2FS6TH7
-A KUBE-SVC-ODX2UBAZM7RQWOIU -j KUBE-SEP-7QEF2MJOSPQ44GT7
```

KUBE-SVC-ODX2UBAZM7RQWOIU是一组规则，是一组随机模式的（-mode random）的iptables链：

1. 第一条目的地是KUBE-SEP-UBFECYXO7WGMGUOV，probability是1/3

   1. ```shell
      [root@rain-kubernetes-1 rain]# iptables-save | grep KUBE-SEP-UBFECYXO7WGMGUOV
      -A KUBE-SEP-UBFECYXO7WGMGUOV -p tcp -m tcp -j DNAT --to-destination 10.168.0.10:9376
      ```

2. 第二条目的地是KUBE-SEP-SGM2NWFJJ2FS6TH7，probability是1/2

   1. ```shell
      [root@rain-kubernetes-1 rain]# iptables-save | grep KUBE-SEP-SGM2NWFJJ2FS6TH7
      -A KUBE-SEP-SGM2NWFJJ2FS6TH7 -p tcp -m tcp -j DNAT --to-destination 10.168.1.12:9376
      ```

3. 第三条目的地是KUBE-SEP-7QEF2MJOSPQ44GT7

   1. ```shell
      [root@rain-kubernetes-1 rain]# iptables-save | grep KUBE-SEP-7QEF2MJOSPQ44GT7
      -A KUBE-SEP-7QEF2MJOSPQ44GT7 -p tcp -m tcp -j DNAT --to-destination 10.168.2.21:9376
      ```

分别对应三个Pod后端，而因为iptables规则的匹配是从上到下逐条进行的，所以为了保证上述三条规则每条被选中的概率都相同：

1. 第一条规则被选中的概率就是 1/3；
2. 而如果第一条规则没有被选中，那么这时候就只剩下两条规则了，所以第二条规则的 probability 就必须设置为 1/2；
3. 类似地，最后一条就必须设置为 1。

> Tips：Service还支持会话亲和性（Session affinity），可以将来自同一个客户端的请求始终转发到同一后端的Pod对象。Session affinity的效果仅在一定时间期限内生效，默认值为10800秒，超出该时长后，客户端再次访问会被调度算法重新调度。Session affinity仅基于客户端的IP地址来识别客户端的身份，会把经由同一个NAT服务器进行原地址转换的客户端识别为同一个客户端，实践中不推荐该方法。通过spec.sessionAffinity, spec.sessionAffinityConfig两个字段来配置。

这三条链都是DNAT规则，DNAT 规则的作用，就是在 PREROUTING 检查点之前，也就是在路由之前，将流入 IP 包的目的地址和端口，改成–to-destination 所指定的新的目的地址和端口。这个目的地址和端口，正是被代理 Pod 的 IP 地址和端口。

![image-20191106163637088](/assets/images/image-20191106163637088.png)

而这些规则的产生，是kube-proxy通过监听Pod的变化事件，在宿主机上生成并维护的。

而除了iptables代理模式，还有userspace代理模式和ipvs代理模式。略。。。

