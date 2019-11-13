---
layout: post
title:  "Kubernetes Job"
date:   2019-11-13 19:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Controller
excerpt: Kubernetes Job
mathjax: true
typora-root-url: ../
---

# Job控制器

* 在线业务：也就是长作业，这些应用一旦运行起来，除非出错或者停止，容器进程会一直在running状态
* 离线业务：也叫作计算业务，这种业务在计算完成后就退出了，Job就是专门描述离线业务的对象

Job也是一种Pod的控制器，用于调配Pod对象运行一次性任务：

* 容器中进程在正常运行结束后不会对其进行重启，而是将Pod对象置于"completed"状态
* 如果容器中进程因错误而终止，则需要依据配置确定重启与否，未运行完成的Pod对象因其所在节点的节点故障而意外终止后会被重新调度

# Job定义

创建一个Job

```yaml
---
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: resouer/ubuntu-bc
        command: ["sh", "-c", "echo 'scale=10000; 4*a(1)' | bc -l "]
      restartPolicy: Never
  backoffLimit: 4
```

Job也是Pod的控制器，当然里面定义的也是Pod模板了

```shell
[root@rain-kubernetes-1 rain]# kubectl get job
NAME   COMPLETIONS   DURATION   AGE
pi     0/1           30s        30s

[root@rain-kubernetes-1 rain]# kubectl describe job pi
Name:           pi
Namespace:      default
Selector:       controller-uid=3a2fff60-a636-4d10-82a0-12bff77c4958
Labels:         controller-uid=3a2fff60-a636-4d10-82a0-12bff77c4958
                job-name=pi
Annotations:    kubectl.kubernetes.io/last-applied-configuration:
                  {"apiVersion":"batch/v1","kind":"Job","metadata":{"annotations":{},"name":"pi","namespace":"default"},"spec":{"backoffLimit":4,"template":...
Parallelism:    1
Completions:    1
Start Time:     Wed, 13 Nov 2019 09:45:57 +0000
Pods Statuses:  1 Running / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  controller-uid=3a2fff60-a636-4d10-82a0-12bff77c4958
           job-name=pi
  Containers:
   pi:
    Image:      resouer/ubuntu-bc
    Port:       <none>
    Host Port:  <none>
    Command:
      sh
      -c
      echo 'scale=10000; 4*a(1)' | bc -l
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From            Message
  ----    ------            ----  ----            -------
  Normal  SuccessfulCreate  50s   job-controller  Created pod: pi-gsdp2
  
  [root@rain-kubernetes-1 rain]# kubectl get pod | grep pi
NAME                                READY   STATUS    RESTARTS   AGE
pi-gsdp2                            1/1     Running   0          101s
```

apply之后，这个Job就创建起来了，job管理的pod也running了，pod的命名是job+一个随机字符串

```shell
[root@rain-kubernetes-1 rain]# kubectl get job
NAME   COMPLETIONS   DURATION   AGE
pi     1/1           2m38s      25m

[root@rain-kubernetes-1 rain]# kubectl get pod | grep pi
pi-gsdp2                            0/1     Completed   0          26m
```

当Job运行完成后，就进入到completed状态，对于Job对象，restartPolicy不可以Always，否则会不断的重启

Job的restartPolicy可以是：

* Never：这种情况下，如果Job执行失败，不会重启Job，会重新创建一个新的Job
  * 重新创建是有次数限制的，spec.backoffLimit来定义重试次数，默认是6
  * 重新创建Pod的间隔呈指数增加，下一次在10s, 20s, 40s ...
* OnFailure：这种情况下，如果Job执行失败，会重启Job

spec.activeDeadlineSeconds字段设置Pod最长运行时间，如果超过了这个运行时间，这个Job的所有Pod都会被终止。并且，你可以在 Pod 的状态里看到终止的原因是 reason: DeadlineExceeded

作业类型：

* 单工作队列串行式Job
  * 以多个一次性作业的方式串行执行多次作业，直至满足期望的次数
  * 在某一个时刻仅存在一个Pod资源对象
* 多工作队列的并行式Job
  * 可以设置工作队列数（作业数），每队列仅负责运行一个作业
  * 或者用有限的工作队列运行较多的作业，工作队列数少于总作业数，相当于运行多个串行作业队列

# Job并行属性

在 Job 对象中，负责并行控制的参数有两个：

1. spec.parallelism
   1. 它定义的是一个 Job 在任意时间最多可以启动多少个 Pod 同时运行，这是并行度属性
   2. parallelism设置为1就是串行方式了
   3. parallelism设置为2以上就是并行多队列运行
2. spec.completions
   1. 它定义的是 Job 至少要完成的 Pod 数目，即 Job 的最小完成数，这是总任务数
   2. completions默认值是1，表示并行度即作业总数
   3. completions大于parallelism，表示使用多队列串行任务作业模式

