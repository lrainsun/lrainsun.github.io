---
layout: post
title:  "Kubernetes CronJob"
date:   2019-11-14 21:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Controller
excerpt: Kubernetes CronJob
mathjax: true
typora-root-url: ../
---

# CronJob

CronJob是管理定时任务的控制器，上次学习了Job是Pod控制器，控制的对象是Pod，而CronJob是Job控制器，控制的对象是Job。CronJob可以：

* 在未来某时间点运行作业一次
* 在指定时间点重复运行作业

````yaml
---
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
````

CronJob的定义里，jobTemplate是必须要的，schedule也是必须要的，也就是job执行的时间

Cron 表达式里 */1 中的 * 表示从 0 开始，/ 表示“每”，1 表示偏移量。

所以，它的意思就是：从 0 开始，每 1 个时间单位执行一次。
Cron 表达式中的五个部分分别代表：分钟、小时、日、月、星期。
上面这句 Cron 表达式的意思是：从当前开始，每分钟执行一次。

```shell
[root@rain-kubernetes-1 rain]# kubectl get cronjob
NAME    SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
hello   */1 * * * *   False     0        47s             23h

[root@rain-kubernetes-1 rain]# kubectl get job | grep hello
hello-1573734780   1/1           6s         2m45s
hello-1573734840   1/1           6s         105s
hello-1573734900   1/1           5s         44s

[root@rain-kubernetes-1 rain]# kubectl get pod | grep hello
hello-1573734780-vnvw2              0/1     Completed   0          2m53s
hello-1573734840-vwvn5              0/1     Completed   0          113s
hello-1573734900-b6mq6              0/1     Completed   0          52s

[root@rain-kubernetes-1 rain]# kubectl logs hello-1573734960-mqsdt
Thu Nov 14 12:36:15 UTC 2019
Hello from the Kubernetes cluster
```

## 并发策略

由于定时任务的特殊性，很可能某个 Job 还没有执行完，另外一个新 Job 就产生了。

可以通过 spec.concurrencyPolicy 字段来定义具体的处理策略。比如：

* concurrencyPolicy=Allow（默认）
  * 意味着这些 Job 可以同时存在
* concurrencyPolicy=Forbid
  * 不会创建新的 Pod，该创建周期被跳过
* concurrencyPolicy=Replace
  * 新产生的 Job 会替换旧的、没有执行完的 Job

## Deadline

如果某一次 Job 创建失败，这次创建就会被标记为“miss”。当在指定的时间窗口内，miss的数目达到 100 时，那么 CronJob 会停止再创建这个 Job。

这个时间窗口，可以由 spec.startingDeadlineSeconds 字段指定。比如startingDeadlineSeconds=200，意味着在过去 200 s 里，如果 miss 的数目达到了 100 次，那么这个 Job 就不会被创建执行了。

## History

```shell
[root@rain-kubernetes-1 rain]# kubectl get job | grep hello
hello-1573734780   1/1           6s         2m45s
hello-1573734840   1/1           6s         105s
hello-1573734900   1/1           5s         44s
```

在get job的时候可以看到三个job，这是`.spec.successfulJobsHistoryLimit` 和 `.spec.failedJobsHistoryLimit` 这两个字段定义的，它们是是可选的。它们指定了可以保留完成和失败 Job 数量的限制。`.spec.successfulJobsHistoryLimit`默认是3，`.spec.failedJobsHistoryLimit` 默认是1

如果不想在执行周期性任务了，那么可以把它挂起

```shell
[root@rain-kubernetes-1 rain]# kubectl patch cronjob hello -p '{"spec":{"suspend":true}}'
cronjob.batch/hello patched

[root@rain-kubernetes-1 rain]# kubectl get cronjob hello
NAME    SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
hello   */1 * * * *   True      0        44s             24h
```



