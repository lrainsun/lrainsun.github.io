---
layout: post
title:  "Kubernetes DaemonSet"
date:   2019-11-15 21:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Controller
excerpt: Kubernetes DaemonSet
mathjax: true
typora-root-url: ../
---

# DaemonSet

DaemonSet也是一种pod控制器，顾名思义，是在cluster所有节点上同时运行一份指定的pod的副本，每个节点有且仅有一个这样的pod。当有新node加入，需要在新节点上也运行一个pod；当有node被删除，对应的pod也要删除。

 有如下三个特征：

1.  这个 Pod 运行在 Kubernetes 集群里的每一个节点（Node）上；
2. 每个节点上只有一个这样的 Pod 实例；
3. 当有新的节点加入 Kubernetes 集群后，该 Pod 会自动地在新节点上被创建出来；而当旧节点被删除后，它上面的 Pod 也相应地会被回收掉。

有如下几种应用场景:

1. 各种网络插件的 Agent 组件，都必须运行在每一个节点上，用来处理这个节点上的容器网络；
2. 各种存储插件的 Agent 组件，也必须运行在每一个节点上，用来在这个节点上挂载远程存储目录，操作容器的 Volume 目录；比如ceph等
3. 各种监控组件和日志组件，也必须运行在每一个节点上，负责这个节点上的监控信息和日志搜集。比如prometheus 的node exporter, collectd, logstash等

# 创建DaemonSet

```yaml
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      containers:
      - name: fluentd-elasticsearch
        image: k8s.gcr.io/fluentd-elasticsearch:1.20
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

跟Deployment很类似，只是没有replicas字段，创建后，每个node上都生成了一个fluentd-elasticsearch的pod

```shell
[root@rain-kubernetes-1 rain]# kubectl get daemonset -n kube-system | grep fluentd-elasticsearch
NAME                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
fluentd-elasticsearch     3         3         3       3            3           <none>                        22s

beta.kubernetes.io/os=linux   23d
[root@rain-kubernetes-1 rain]# kubectl get pod -n kube-system | grep fluentd
fluentd-elasticsearch-p52jf                 1/1     Running   0          40s
fluentd-elasticsearch-q94cn                 1/1     Running   0          40s
fluentd-elasticsearch-xdk6j                 1/1     Running   0          40s
```

# DaemonSet Controller

DaemonSet controller要做的事情就是，保证每个node上有且仅有一个pod

1. 没有这种 Pod，那么就意味着要在这个 Node 上创建这样一个 Pod；
2. 有这种 Pod，但是数量大于 1，那就说明要把多余的 Pod 从这个 Node 上删除掉；
3. 正好只有一个这种 Pod，那说明这个节点是正常的

怎样在一个node上创建一个pod呢？这是由daemonset controller生成的pod的信息

```yaml
[root@rain-kubernetes-1 rain]# kubectl get pod fluentd-elasticsearch-p52jf -n kube-system -o yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2019-11-15T13:58:29Z"
  generateName: fluentd-elasticsearch-
  labels:
    controller-revision-hash: 5d8f4f6fc8
    name: fluentd-elasticsearch
    pod-template-generation: "1"
  name: fluentd-elasticsearch-p52jf
  namespace: kube-system
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: DaemonSet
    name: fluentd-elasticsearch
    uid: 3247cba5-0791-4c68-987d-78d7a9cfd0e7
  resourceVersion: "2986872"
  selfLink: /api/v1/namespaces/kube-system/pods/fluentd-elasticsearch-p52jf
  uid: dd61f054-9ad5-4709-b5b3-b7244999bfc1
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchFields:
          - key: metadata.name
            operator: In
            values:
            - rain-kubernetes-2
  containers:
  - image: k8s.gcr.io/fluentd-elasticsearch:1.20
    imagePullPolicy: IfNotPresent
    name: fluentd-elasticsearch
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/log
      name: varlog
    - mountPath: /var/lib/docker/containers
      name: varlibdockercontainers
      readOnly: true
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-m2slm
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: rain-kubernetes-2
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
  - effect: NoSchedule
    key: node.kubernetes.io/disk-pressure
    operator: Exists
  - effect: NoSchedule
    key: node.kubernetes.io/memory-pressure
    operator: Exists
  - effect: NoSchedule
    key: node.kubernetes.io/pid-pressure
    operator: Exists
  - effect: NoSchedule
    key: node.kubernetes.io/unschedulable
    operator: Exists
```

有两个小技巧：

* 当controller发现一个node上没有pod，想要在这个node上创建一个pod的时候，是如何指定node的呢？

```yaml
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchFields:
          - key: metadata.name
            operator: In
            values:
            - rain-kubernetes-2
```

用的是nodeAffinity的属性，通过match metadata.name字段。requiredDuringSchedulingIgnoredDuringExecution：它的意思是说，这个nodeAffinity 必须在每次调度的时候予以考虑。

* 如果我们想要部署的daementset是network agent，那么在网络不通之前，是会有"node.kubernetes.io/unreachable"这样的污点的，那就不会让pod调度过去，可是我们需要pod调度过去。那么就需要加上这样一个可容忍的标记。通过这样的标记，scheduler就不会care这个，而让pod跑在这样的node上了

```yaml
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
```

# 版本管理

之前学习 [无状态应用](https://lrainsun.github.io/2019/11/03/kubernetes-stateless-applications/) 的时候，“一个版本对应一个 ReplicaSet 对象”，来进行版本控制。那daemonset这种版本管理怎么弄呢？

先试试，更新一下image，看看状态

```shell
[root@rain-kubernetes-1 rain]# kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=k8s.gcr.io/fluentd-elasticsearch:1.21 -n kube-system
daemonset.apps/fluentd-elasticsearch image updated

[root@rain-kubernetes-1 rain]# kubectl get daemonset fluentd-elasticsearch -n kube-system
NAME                    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
fluentd-elasticsearch   3         3         2       1            2           <none>          28m

[root@rain-kubernetes-1 rain]# kubectl rollout status daemonset fluentd-elasticsearch -n kube-system
Waiting for daemon set "fluentd-elasticsearch" rollout to finish: 1 out of 3 new pods have been updated...

[root@rain-kubernetes-1 rain]# kubectl rollout history daemonset fluentd-elasticsearch -n kube-system
daemonset.apps/fluentd-elasticsearch
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

在更新了，也检查得到rollout status，deployment使用replicaset控制版本，可是daemonset直接控制的是pod，怎么做到版本管理呢？

一切皆对象！
在 Kubernetes 项目中，任何你觉得需要记录下来的状态，都可以被用 API 对象的方式实现。
“版本”也不例外，Kubernetes v1.7 之后添加了一个 API 对象，名叫ControllerRevision，专门用来记录某种Controller 对象的版本

```shell
[root@rain-kubernetes-1 rain]# kubectl get controllerrevision -n kube-system | grep fluentd-elasticsearch
fluentd-elasticsearch-57674b5587     daemonset.apps/fluentd-elasticsearch     2          17m
fluentd-elasticsearch-5d8f4f6fc8     daemonset.apps/fluentd-elasticsearch     1          44m
```

用起来其实差不多的