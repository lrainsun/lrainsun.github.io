---
layout: post
title:  "Kubernetes里的有状态应用"
date:   2019-11-16 18:03:00 +0800
categories: Kubernetes
tags: Kubernetes-Controller
excerpt: Kubernetes里有状态应用，StatefulSet
mathjax: true
typora-root-url: ../
---

# 有状态应用

Pod的管理对象，比如ReplicaSet，Deployment，DaremonSet和Job等都是面向无状态的服务。

它们认为，一个应用的所有 Pod，是完全一样的。所以，它们互相之间没有顺序，也无所谓运行在哪台宿主机上。需要的时候，Deployment 就可以通过 Pod 模板创建新的 Pod；不需要的时候，Deployment 就可以“杀掉”任意一个 Pod。

但是现实中有很多服务是有状态的，尤其是分布式应用，它的多个实例之间，往往有依赖关系，比如：主从关系、主备关系。还有就是数据存储类应用，它的多个实例，往往都会在本地磁盘上保存一份数据。而这些实例一旦被杀掉，即便重建出来，实例与数据之间的对应关系也已经丢失，从而导致应用失败。

所以，这种实例之间有不对等关系，以及实例对外部数据有依赖关系的应用，就被称为“有状态应用”（Stateful Application）。

StatefulSet就是用来支持这样的有状态应用的。

一直回避StatefulSet，因为有点晕，尤其是有个mysql集群的例子，倒也不是完全看不懂，但总是看得有点费劲。感觉如果真是比较复杂的处理逻辑，是不是应该用operator来实现更好，或者operator里调用statefulset。先不纠结那么多，先来看看statefulset的概念

# StatefulSet

最关键的两点，它把真实世界里的应用状态，抽象为了两种情况：

1. ***拓扑状态***。这种情况意味着，应用的多个实例之间不是完全对等的关系。这些应用实例，必须按照某些顺序启动，比如应用的主节点 A 要先于从节点 B 启动。而如果你把 A 和 B 两个Pod 删除掉，它们再次被创建出来时也必须严格按照这个顺序才行。并且，新创建出来的Pod，必须和原来 Pod 的网络标识一样，这样原先的访问者才能使用同样的方法，访问到这个新 Pod。
2. ***存储状态***。这种情况意味着，应用的多个实例分别绑定了不同的存储数据。对于这些应用实例来说，Pod A 第一次读取到的数据，和隔了十分钟之后再次读取到的数据，应该是同一份，哪怕在此期间 Pod A 被重新创建过。这种情况最典型的例子，就是一个数据库应用的多个存储实例。

StatefulSet 的核心功能，就是通过某种方式记录这些状态，然后在 Pod 被重新创建时，能够为新 Pod 恢复这些状态。

## 拓扑状态

之前[DNS](https://lrainsun.github.io/2019/11/09/kubernetes-dns/)里说过headless service，headless service会把一组pod暴露给外部，由client自己来做这个负载均衡。headless service是没有clusterip的，靠service的dns（`my-svc.my-namespace.svc.cluster.local` ）来访问。同样，headless service里的每个pod，也都有一条dns记录对应着每一个pod（`my-pod.my-svc.my-namespace.svc.cluster.local`）

也就是说，通过pod name + service name，就可以获取到pod的ip。statefulset正是利用了这一点，维护了pod的拓扑状态。

先创建一个headless service

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
```

这个时候并没有app=nginx的pod存在，所以这个service还没有任何endpoint

```shell
[root@rain-kubernetes-1 rain]# kubectl get service nginx
NAME    TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
nginx   ClusterIP   None         <none>        80/TCP    8m54s

[root@rain-kubernetes-1 rain]# kubectl describe service nginx
Name:              nginx
Namespace:         default
Labels:            app=nginx
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app":"nginx"},"name":"nginx","namespace":"default"},"spec":{"c...
Selector:          app=nginx
Type:              ClusterIP
IP:                None
Port:              web  80/TCP
TargetPort:        80/TCP
Endpoints:         <none>
Session Affinity:  None
Events:            <none>
```

这个时候我们创建一个statefulset

```yaml
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"  #指定了headless service的名字
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web
```

这个statefulset的定义跟deployment基本一致，多了一个serviceName=nginx字段。

这个字段的作用，就是告诉 StatefulSet 控制器，在执行控制循环（Control Loop）的时候，请使用 nginx 这个 Headless Service 来保证 Pod 的“可解析身份”。

```shell
[root@rain-kubernetes-1 rain]# kubectl get statefulset web
NAME   READY   AGE
web    2/2     4m12s

[root@rain-kubernetes-1 rain]# kubectl get pod -l app=nginx
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          5m29s
web-1   1/1     Running   0          5m14s

[root@rain-kubernetes-1 rain]# kubectl describe service nginx
Name:              nginx
Namespace:         default
Labels:            app=nginx
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app":"nginx"},"name":"nginx","namespace":"default"},"spec":{"c...
Selector:          app=nginx
Type:              ClusterIP
IP:                None
Port:              web  80/TCP
TargetPort:        80/TCP
Endpoints:         10.168.0.93:80,10.168.1.184:80
Session Affinity:  None
Events:            <none>
```

statefulset创建出了两个pod，他们的名字是有编号的，而且，创建顺序是严格按照顺序进行的。web-0一定比web-1先创建，在web-0没running之前，web-1会一直在pending状态。两个pod的hostname跟它们的pod name是一致的。

试一下pod的dns

```shell
[root@rain-kubernetes-1 rain]# kubectl run dig --rm -it --image=docker.io/azukiapp/dig /bin/sh
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
If you don't see a command prompt, try pressing enter.
/ # nslookup web-0.nginx
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   web-0.nginx.default.svc.cluster.local
Address: 10.168.1.184

/ # nslookup web-1.nginx
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   web-1.nginx.default.svc.cluster.local
Address: 10.168.0.93
```

试一下把这两个pod删除

```shell
[root@rain-kubernetes-1 rain]# kubectl delete pod -l app=nginx
pod "web-0" deleted
pod "web-1" deleted

[root@rain-kubernetes-1 rain]# kubectl get pod -l app=nginx
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          5s
web-1   1/1     Running   0          2s

[root@rain-kubernetes-1 rain]# kubectl run dig --rm -it --image=docker.io/azukiapp/dig /bin/sh
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
If you don't see a command prompt, try pressing enter.
/ # nslookup web-0.nginx
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   web-0.nginx.default.svc.cluster.local
Address: 10.168.2.125

/ # nslookup web-1.nginx
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   web-1.nginx.default.svc.cluster.local
Address: 10.168.1.185
```

web-0依然比web-1先创建，虽然两个pod的ip都变了，可是他们的dns并没有变。这样就可以将pod的拓扑状态（哪个节点先启动，哪个节点后启动）按照pod的“名字+编号"固定下来了。

这些状态，在 StatefulSet 的整个生命周期里都会保持不变，绝不会因为对应 Pod 的删除或者重新创建而失效。

尽管 web-0.nginx 这条记录本身不会变，但它解析到的 Pod 的IP 地址，并不是固定的。这就意味着，对于“有状态应用”实例的访问，你必须使用 DNS 记录或者 hostname 的方式，而绝不应该直接访问这些 Pod 的 IP 地址。

## 存储状态

之前pod的时候，学习过projected volume，除了这个，还可以定义hostPath的 volume，除此之外，还有PVC和PV的概念。正如statefulset对拓扑状态的管理依赖于headless service一样，statefulset对存储状态的管理需要依赖于持久化存储。

这其中PVC（persistent volume claim）和PV（persistent volume），先不去管PVC的概念，PVC和PV就像是，接口和实现，我们只要知道并使用PVC（接口)，而管理员把负责把PV定义好，我们只管定义PVC就行了。PVC可以理解为一种特殊的volume，定义了PVC，我们就有了一个持久化的存储（别管他是啥类型，PVC跟具体的PV绑定后才能知道）。

就像statefulset里的podtemplate一样，statefulset里的持久化存储的定义也有一个volumeclaimtemplates。

statefulset里的pod不是有固定的拓扑状态么，同样，statefulset里的pvc也有同样的拓扑状态。每个Pod都会对应有一个同样编号的PVC，而这个同样编号的PVC（跟PV绑定后，进入Bound状态，就可以被挂载了）会挂载到同样编号的Pod上。

我们实现事先准备好两个nfs pv

```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 10.225.28.220
    path: /root/nfs
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv1
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 10.225.28.220
    path: /root/nfs1
```

看下当前状态，都是available的

```shell


[root@rain-kubernetes-1 rain]# kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
nfs-pv    100Mi      RWO            Retain           Available                                   6s
nfs-pv1   100Mi      RWO            Retain           Available                                   6s
```

在刚刚的statefulset里增加一个volumeclaintemplates的定义，同时为每个pod挂载一个volume

```yaml
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Mi
```

再看下当前pv, pvc, pod, statefulset的状态

```shell
[root@rain-kubernetes-1 rain]# kubectl get statefulset web
NAME   READY   AGE
web    2/2     12s

[root@rain-kubernetes-1 rain]# kubectl get pod -l app=nginx
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          21s
web-1   1/1     Running   0          17s

[root@rain-kubernetes-1 rain]# kubectl get pvc
NAME        STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
www-web-0   Bound    nfs-pv    100Mi      RWO                           30s
www-web-1   Bound    nfs-pv1   100Mi      RWO                           26s

[root@rain-kubernetes-1 rain]# kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
nfs-pv    100Mi      RWO            Retain           Bound    default/www-web-0                           3m
nfs-pv1   100Mi      RWO            Retain           Bound    default/www-web-1                           3m
```

有两个pvc被创建出来了，www-web-0 & www-web-1，以“<PVC 名字 >-<StatefulSet 名字 >-< 编号 >”的方式命名，并且处于 Bound 状态。

这个编号跟pod编号是一致的，pod web-0会使用pvc www-web-0，pod web-1会使用pvc www-web-1

而如果删除两个pod，会发生什么呢？

```shell
[root@rain-kubernetes-1 rain]# kubectl delete pod -l app=nginx
pod "web-0" deleted
pod "web-1" deleted

[root@rain-kubernetes-1 rain]# kubectl get pvc
NAME        STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
www-web-0   Bound    nfs-pv    100Mi      RWO                           7m33s
www-web-1   Bound    nfs-pv1   100Mi      RWO                           7m29s
```

pvc并不会被删除，也就说数据是会被留着的。过一会儿这两个pod又会被按照编号顺序重新创建出来，pvc www-web-0依然会被绑定到web-0，pvc www-web-1依然会被绑定到web-1，这个绑定关系不会变。

# 工作原理

不得不摘抄一下了

StatefulSet 的设计思想：StatefulSet 其实就是一种特殊的Deployment，而其独特之处在于，它的每个 Pod 都被编号了。而且，这个编号会体现在 Pod的名字和 hostname 等标识信息上，这不仅代表了 Pod 的创建顺序，也是 Pod 的重要网络标识（即：在整个集群里唯一的、可被的访问身份）。
有了这个编号后，StatefulSet 就使用 Kubernetes 里的两个标准功能：Headless Service 和PV/PVC，实现了对 Pod 的拓扑状态和存储状态的维护。

首先，StatefulSet 的控制器直接管理的是 Pod。这是因为，StatefulSet 里的不同 Pod 实例，不再像 ReplicaSet 中那样都是完全一样的，而是有了细微区别的。比如，每个 Pod 的hostname、名字等都是不同的、携带了编号的。而 StatefulSet 区分这些实例的方式，就是通过在 Pod 的名字里加上事先约定好的编号。
其次，Kubernetes 通过 Headless Service，为这些有编号的 Pod，在 DNS 服务器中生成带有同样编号的 DNS 记录。只要 StatefulSet 能够保证这些 Pod 名字里的编号不变，那么 Service里类似于 web-0.nginx.default.svc.cluster.local 这样的 DNS 记录也就不会变，而这条记录解析出来的 Pod 的 IP 地址，则会随着后端 Pod 的删除和再创建而自动更新。这当然是 Service机制本身的能力，不需要 StatefulSet 操心。
最后，StatefulSet 还为每一个 Pod 分配并创建一个同样编号的 PVC。这样，Kubernetes 就可以通过 Persistent Volume 机制为这个 PVC 绑定上对应的 PV，从而保证了每一个 Pod 都拥有一个独立的 Volume。
在这种情况下，即使 Pod 被删除，它所对应的 PVC 和 PV 依然会保留下来。所以当这个 Pod被重新创建出来之后，Kubernetes 会为它找到同样编号的 PVC，挂载这个 PVC 对应的Volume，从而获取到以前保存在 Volume 里的数据。