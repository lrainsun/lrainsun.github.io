---
layout: post
title:  "Kubernetes里的无状态应用"
date:   2019-11-03 22:03:00 +0800
categories: Kubernetes
tags: Kubernetes-Controller
excerpt: Kubernetes里无状态应用，deployment, repliaset
mathjax: true
typora-root-url: ../
---

# Deployment

Deployment是kubernetes控制器的一种实现，而控制器是kubernetes项目里最重要的设计思想之一。控制器的基本原理后面也值得细细研究一下。

Deployment通过使用ReplicaSets，实现了这样一些功能：

1. 事件和状态查看：必要时可以查看deployment对象升级的详细进度和状态
2. 回滚：升级操作完成后发现问题时，支持使用回滚机制将应用返回到前一个或由用户指定的历史记录的版本上
3. 版本记录：对deployment对象的每一次操作都予以保存，以供后续可能执行的回滚操作使用
4. 暂停和启动：对于每一次升级，都能够随时暂停和启动
5. 多种自动更新方案：一是Recreate，即重建更新机制，全面停止、删除旧有的Pod后用新版替代；另一种是RollingUpdaet，即滚动升级机制，逐步替换旧有的Pod至新的版本

![这里写图片描述](/assets/images/99adaa8e91b0a0e7c485ecdf156e77b5.png)

replicaset通过"控制器模式"，保证系统中Pod的个数永远等于指定的个数

deployment也通过“控制器模式”，操作replicaset的个数和属性，进而实现"水平扩展/收缩”和"滚动更新"这两个编排动作

# 创建

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
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
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

apply之后

```
[root@rain-kubernetes-1 rain]# kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           3m13s

[root@rain-kubernetes-1 rain]# kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-54f57cf6bf   3         3         3       3m15s

[root@rain-kubernetes-1 rain]# kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-54f57cf6bf-4ljfl   1/1     Running   0          3m15s
nginx-deployment-54f57cf6bf-4vlzl   1/1     Running   0          3m17s
nginx-deployment-54f57cf6bf-wjvmx   1/1     Running   0          3m14s
```

三个pod，一个replicaset，一个deployment被创建了出来，可以注意到三者name，rs名字是在deployment之后带上一个hash，而pod名字是在rs之后带上一个hash，三者之间是有紧密关系的

而rs有：

* DESIRED：用户期望的pod副本的个数
* CURRENT：当前处于running的pod的个数
* READY：当前可用（健康检查正确）的pod个数

deployment：

* READY
* UP-TO-DATE：当前处于最新版本的pod的个数
* AVAILABLE：当前已经可用的pod的个数，既是running状态，又是最新版本，并且已经处于ready的

# 水平扩展/收缩

“水平扩展 / 收缩”非常容易实现，Deployment Controller 只需要修改它所控制的ReplicaSet 的 Pod 副本个数就可以了。比如，把这个值从 3 改成 4，那么 Deployment 所对应的 ReplicaSet，就会根据修改后的值自动创建一个新的 Pod。这就是“水平扩展”了；“水平收缩”则反之。

然后把Replicas从3改成4：

```
[root@rain-kubernetes-1 rain]# kubectl scale deployment nginx-deployment --replicas=4
deployment.apps/nginx-deployment scaled
```

看一下的变化：

```
[root@rain-kubernetes-1 rain]# kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   4/4     4            4           11m

[root@rain-kubernetes-1 rain]# kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-54f57cf6bf   4         4         4       11m

[root@rain-kubernetes-1 rain]# kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-54f57cf6bf-4ljfl   1/1     Running   0          11m
nginx-deployment-54f57cf6bf-4vlzl   1/1     Running   0          11m
nginx-deployment-54f57cf6bf-bx48p   1/1     Running   0          31s
nginx-deployment-54f57cf6bf-wjvmx   1/1     Running   0          11m
```

# 滚动更新

在测试之前，先describe一下deployment：

```
[root@rain-kubernetes-1 rain]# kubectl describe deployment nginx-deployment
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Sun, 03 Nov 2019 08:54:31 +0000
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision: 1
                        kubectl.kubernetes.io/last-applied-configuration:
                          {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"labels":{"app":"nginx"},"name":"nginx-deployment","namespace":"d...
Selector:               app=nginx
Replicas:               4 desired | 4 updated | 4 total | 4 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx:1.7.9
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-deployment-54f57cf6bf (4/4 replicas created)
```

这边StrategyType是rollingupdate（默认）：是在删除一部分旧版本pod的同时，补充创建一部分新版本的pod进行应用升级，不过在更新操作期间，不同客户端得到的应用内容可能会来自不同版本的应用。

另一种type是recreate，通常只在新旧版本不兼容时才会采用recreate策略，会导致应用替换期间暂时不可用，好处在于它不存在中间状态，用户访问的要么是应用的新版本，要么是旧版本。

另外在rollingupdate的时候，还有两个参数可以关注，RollingUpdateStrategy:  25% max unavailable, 25% max surge

maxSurge：除了desired数量之外，在一次rollingupdate的过程中，deployment控制器还可以创建多少新pod，可以是0，正整数，或者一个百分比

maxUnavailable：在一次rollingupdate的过程中，deployment控制器还可以删除多少旧pod，可以是0（跟maxSurge不能同时为0），正整数，或者一个百分比

rollingupdate的时候，指定一个--record参数，就可以记录下每次操作所执行的命令，以方便后面查看

我们试着把nginx版本从1.7.9改到1.8试一下

```
[root@rain-kubernetes-1 rain]# kubectl apply -f 17-deployment.yaml --record
deployment.apps/nginx-deployment configured

[root@rain-kubernetes-1 rain]# kubectl rollout status deployment nginx-deployment
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "nginx-deployment" successfully rolled out

[root@rain-kubernetes-1 rain]# kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           172m

[root@rain-kubernetes-1 rain]# kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-54f57cf6bf   0         0         0       172m
nginx-deployment-9f46bb5      3         3         3       20s

[root@rain-kubernetes-1 rain]# kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
nginx-deployment-9f46bb5-7rxck    1/1     Running   0          23s
nginx-deployment-9f46bb5-ffg97    1/1     Running   0          22s
nginx-deployment-9f46bb5-mw6rg    1/1     Running   0          20s
```

可以看到，生成了一个新的replicaset，但是旧的replicaset都是0，但还是保留了

一次rollingupdate，revision就+1

```
[root@rain-kubernetes-1 rain]# kubectl describe deployment nginx-deployment
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Sun, 03 Nov 2019 08:54:31 +0000
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision: 2
```

再把image设置成不存在的1.19

```
[root@rain-kubernetes-1 rain]# kubectl rollout history deployment/nginx-deployment
deployment.apps/nginx-deployment
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl apply --filename=17-deployment.yaml --record=true
3         kubectl set image deployment/nginx-deployment nginx=nginx:1.91 --record=true
```

# Rollback

试一下rollback，undo到上一个revision

```
[root@rain-kubernetes-1 rain]# kubectl rollout undo deployment/nginx-deployment
deployment.apps/nginx-deployment rolled back

[root@rain-kubernetes-1 rain]# kubectl rollout history deployment/nginx-deployment
deployment.apps/nginx-deployment
REVISION  CHANGE-CAUSE
1         <none>
3         kubectl set image deployment/nginx-deployment nginx=nginx:1.91 --record=true
4         kubectl apply --filename=17-deployment.yaml --record=true

[root@rain-kubernetes-1 rain]# kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-54f57cf6bf   0         0         0       3h18m
nginx-deployment-84fdd4f88c   0         0         0       12m
nginx-deployment-9f46bb5      3         3         3       26m
```

rollback也会生成一个新的revision

除此之外，还可以rollback到指定revision

```
[root@rain-kubernetes-1 rain]# kubectl rollout undo deployment/nginx-deployment --to-revision=1
deployment.apps/nginx-deployment rolled back

[root@rain-kubernetes-1 rain]# kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-54f57cf6bf   3         3         3       3h22m
nginx-deployment-84fdd4f88c   0         0         0       15m
nginx-deployment-9f46bb5      0         0         0       30m
```

当回到revision=1的时候，用的仍然是最初的replicaset

如果想要不生成新的replicaset，可以在更新 Deployment 前，先执行一条 kubectl rollout pause 指令，等到更新都完成后，再执行一条kubectl rollout resume。或者在Deployment 对象有一个字段，叫作spec.revisionHistoryLimit，就是 Kubernetes为 Deployment 保留的“历史版本”个数。所以，如果把它设置为 0，你就再也不能做回滚操作了。