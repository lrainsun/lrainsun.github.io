---
layout: post
title:  "Kubernetes Deployment"
date:   2020-05-11 23:00:00 +0800
categories: Kubernetes
tags: Kubernetes-Controller
excerpt: Kubernetes Deployment
mathjax: true
typora-root-url: ../
---

# Kubernetes Deployment

白天都在跟airflow奋斗，晚上缓慢复习ing

Deployment并不直接管理Pod，而是通过ReplicaSet，ReplicaSet再去管理Pod

![image-20200511215307224](/../assets/images/image-20200511215307224.png)

*  Deployment 只允许容器的 restartPolicy=Always：ReplicaSet 负责通过“控制器模式”，保证系统中 Pod 的个数永远等于指定的个数。只有在容器能保证自己始终是 Running 状态的前提下，ReplicaSet 调整 Pod 的个数才有意义

# rolling update

不管是修改副本数，或者是修改Image等，都会触发rolling update

* rolling update的本质是根据修改后的pod模板，创建一个新的replicaset
* 一边减少旧replicaset里的pod数目，一边增加新replicaset里的pod数，交替地逐一升级
* RollingUpdateStrategy配置定义了update策略：
  * maxSurge：除了desired数量之外，deployment控制器还可以创建多少新pod（可以是个数，也可以是百分比，默认25%）
  * maxUnavailable：在一次滚动中，deployment控制器可以删除多少旧pod（可以是个数，也可以是百分比，默认25%）

# rollout历史

回滚

```
kubectl rollout undo deployment/nginx-deployment
```

历史

```
kubectl rollout history
```

需要在create deployment的时候指定--record，记录下你每次操作所执行的命令，以方便后面查看

```
kubectl create -f nginx-deployment.yaml --record
```

回滚到对应版本

```
kubectl rollout history deployment/nginx-deployment --revision=2
```

Deployment 对象有一个字段，叫作 spec.revisionHistoryLimit，就是 Kubernetes为 Deployment 保留的“历史版本”个数。所以，如果把它设置为 0，你就再也不能做回滚操作了。