---
layout: post
title:  "Kubernetes资源限制"
date:   2019-11-01 22:03:00 +0800
categories: Kubernetes
tags: Kubernetes-Architecture
excerpt: Kubernetes资源限制
mathjax: true
typora-root-url: ../
---

# cpuset

之前有学习过[https://lrainsun.github.io/2019/11/22/kubernetes-resource-management/](https://lrainsun.github.io/2019/11/22/kubernetes-resource-management/)kubernetes资源管理。除此之外，还有一种cpuset的设置，你可以通过设置 cpuset 把容器绑定到某个 CPU 的核上，而不是像 cpushare 那样共享 CPU 的计算能力。

这种情况下，由于操作系统在 CPU 之间进行上下文切换的次数大大减少，容器里应用的性能会得到大幅提升。

* Pod要是Guaranteed的 QoS 类型
* 将 Pod 的 CPU 资源的 requests 和 limits 设置为同一个相等的整数值

# kubernetes资源限制

但是假如应用自己设置很大的limit/request值，那不是把系统资源都用完了吗？如果应用没有设置limit/request又怎么办呢？

## LimitRange

可以用LimitRange定义一个全局的默认配置

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: rain-limit-range
spec:
  limits:
    - max:
        cpu: 2
        memory: 2Gi
      min:
        cpu: 200m
        memory: 6Mi
      maxLimitRequestRatio:
        cpu: 3
        memory: 2
      type: Pod
    - default:
        cpu: 300m
        memory: 200Mi
      defaultRequest:
        cpu: 200m
        memory: 100Mi
      max:
        cpu: 2
        memory: 1Gi
      min:
        cpu: 100m
        memory: 3Mi
      maxLimitRequestRatio:
        cpu: 5
        memory: 4
      type: Container
```

```shell
[root@rain-kubernetes-1 rain]# kubectl apply -f limit-range.yml
limitrange/rain-limit-range created
```

对于pod和container都可以指定：

* `min`：指定资源的下限，即最低资源配置不能低于这个值；
* `max`：指定资源的上限，即最高资源使用不能高于这个值；
* `maxLimitRequestRatio`：该值用于指定requests和limits值比例的上限。

container除此之外还可以指定：

* `defaultRequest`：全局容器的默认`requests`值；

* ` default`：全局容器的默认`limits`值。

测试一下，创建一个没有指定limit/request的Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-limit
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
```

describe一下，发现limit/request被设置成了上面定义的默认值

```yaml
    Limits:
      cpu:     300m
      memory:  200Mi
    Requests:
      cpu:        200m
      memory:     100Mi
```

改一个不在范围内的值试试

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-limit
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
      resources:
        limits:
          cpu: 50m
```

apply之后会报错

```shell
[root@rain-kubernetes-1 rain]# kubectl apply -f limit.yml
The Pod "test-limit" is invalid:
* spec.containers[0].resources.requests: Invalid value: "200m": must be less than or equal to cpu limit
```

再试一下，看看比例如果超过定义值了呢？

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-limit
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
      resources:
        limits:
          cpu: 500m
        requests:
          cpu: 100m
```

```shell
[root@rain-kubernetes-1 rain]# kubectl apply -f limit.yml
Error from server (Forbidden): error when creating "limit.yml": pods "test-limit" is forbidden: [minimum cpu usage per Pod is 200m, but request is 100m, cpu max limit to request ratio per Pod is 3, but provided ratio is 5.000000]
```

## ResourceQuota

ResourceQuota用于管理资源配额，这个资源配额可以为每个命名空间都提供一个总体的资源使用的限制。

我们定义一个resourcequota

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: quota-ns

---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: test-quota
spec:
  hard:
    pods: "2"
```

测试一下：

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-rc
  namespace: quota-ns
spec:
  replicas: 3
  selector:
    name: nginx
  template:
    metadata:
      labels:
        name: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
```

replica里定义了3个pod，实际我们看跑起来是2个，不会到3个，因为最大限制pod数是2

```shell
[root@rain-kubernetes-1 rain]# kubectl get pods -n quota-ns
NAME             READY   STATUS    RESTARTS   AGE
nginx-rc-cplqk   1/1     Running   0          68s
nginx-rc-lvmpd   1/1     Running   0          68s
```

试了一下，如果本来跑了3个pod，再去把限制改为2，pod数不会减少，也就是，只会影响设置了ResourceQuota之后的

