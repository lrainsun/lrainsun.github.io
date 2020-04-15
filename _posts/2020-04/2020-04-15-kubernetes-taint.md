---
layout: post
title:  "kubernetes中的Taint和Toleration"
date:   2020-04-15 23:30:00 +0800
categories: Kubernetes
tags: Kubernetes-Scheduler
excerpt: kubernetes中的Taint和Toleration
mathjax: true
---

# kubernetes中的Taint和Toleration

一个Pod启动可以调度到哪个node上是有一个选择的过程，这中间有一个把某些node排除在外的方法，就是给node打上Taint，Taint的意思是污点，意思很容易理解就是，当一个node被打上了污点了之后，Pod就看不上了（洁癖）。除非有Pod说，我不是那么介意这个污点，我对这个污点可以容忍Toleration，那么这个Pod才有可能被调度到打上污点的Node。

# Taint

一个Taint的定义分为三个部分：

* key
* value
* effect

key和value很好理解，就是平时说的键值对，也可以只定义key而没有value

effect共有三种

* NoSchedule：假如某个node上定义的taint的effect是NoSchedule，那kubernetes就肯定不把Pod调度到这个node上，但不影响已经运行的Pod 
* PreferNoSchedule：假如某个node上定义的taint的effect是PreferNoSchedule，那kubernetes就尽量不把Pod调度到这个node上（但如果实在找不到其他node，还是有可能调度过去的），不影响已经运行的Pod 
* NoExecute：假如某个node上定义的taint的effect是NoExecute，那kubernetes就肯定不把Pod调度到这个node上，而且kubernetes还会把已经存在该node上的Pod赶走

比如这样一个taint

```shell
kubectl taint nodes node1 key=value:NoSchedule
```

如果要删除taint

```shell
kubectl taint nodes node1 key:NoSchedule-
```

# Toleration

```yaml
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
```

```yaml
tolerations:
- key: "key"
  operator: "Exists"
  effect: "NoSchedule"
```

这两个toleration的定义都可以容忍上面的污点：

* 一个 toleration 和一个 taint 相“匹配”是指它们有一样的 key 和 effect ，并且：
  - 如果 `operator` 是 `Exists` （此时 toleration 不能指定 `value`），或者
  - 如果 `operator` 是 `Equal` ，则它们的 `value` 应该相等

两种特殊情况：

- 如果一个 toleration 的 `key` 为空且 operator 为 `Exists` ，表示这个 toleration 与任意的 key 、 value 和 effect 都匹配，即这个 toleration 能容忍任意 taint。

  ```yaml
  tolerations:
  - operator: "Exists"
  ```

- 如果一个 toleration 的 `effect` 为空，则 `key` 值与之相同的相匹配 taint 的 `effect` 可以是任意值。

  ```yaml
  tolerations:
  - key: "key"
  operator: "Exists"
  ```

