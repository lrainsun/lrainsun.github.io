---
layout: post
title:  "Kubernetes Monitor - Prometheus operator"
date:   2019-11-28 23:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Monitor
excerpt: Kubernetes Monitor - Prometheus operator
mathjax: true
typora-root-url: ../
---

# Kubernetes监控

Kubernetes监控是以prometheus项目为核心的一套统一的方案。

![架构](/assets/images/prometheus-architecture.png)

Prometheus 由多个组件组成：

- Prometheus Server：用于抓取指标、存储时间序列数据
- exporter：暴露指标让任务来抓
- pushgateway：push 的方式将指标数据推送到该网关
- alertmanager：处理报警的报警组件
- adhoc：用于数据查询

Prometheus 直接接收或者通过中间的 Pushgateway 网关被动获取指标数据，在本地存储所有的获取的指标数据，并对这些数据进行一些规则整理，用来生成一些聚合数据或者报警信息，Grafana 或者其他工具用来可视化这些数据。

# Metrics

对kubernetes的监控大概可以分为以下几种类型的metrics：

* 宿主机的监控数据：

  * node exporter（DaementSet）

* kubernetes API Server, kubelet的/metrics API

  * 各个组件的核心监控指标

    * 因为--listen-metrics-urls=http://127.0.0.1:2381

      所以在我的环境里 `curl http://127.0.0.1:2381/metrics` 可以get到etcd的metrics

    * `kubectl get --raw /metrics` kube-apiserver的metrics

* kubernetes相关的监控数据（kubernetes核心监控数据core metrics）

  * pod, node, container（内置cAdvisor服务）, service等主要kubernetes核心概念的metrics 

# Prometheus-Operator

为什么要引入promethus operator来监控kubernetes呢？

* kubernetes里pod因为调度的原因导致pod的ip会发生变化
* prometheus server, alertmanager的高可用，prometheus可以通过服务发现的形式来自动监控集群

![promtheus opeator](/assets/images/prometheus-operator.png)

Operator的关键是CRD（自定义资源）的设计，prometheus-operator里定义了5个CRD：

* Prometheus：作为prometheus server存在
* ServiceMonitor：exportor的各种抽象，描述了一组被 Prometheus 监控的目标列表
* AlertManager：对应的AlertManager的抽象
* PrometheusRule： 被Prometheus实例使用的报警规则文件
* PodMonitor

Operator作为一个控制器，会创建上面这四种CRD资源对象，然后会一直监控并维持他们的状态。

一个 ServiceMonitor 可以通过 labelSelector 的方式去匹配一类 Service，Prometheus 也可以通过 labelSelector 去匹配多个ServiceMonitor。

# 手工部署prometheus-operator

为加深理解，我们手工部署Prometheus-operator

## clone源代码

```shell
git clone https://github.com/coreos/prometheus-operator.git
```

## 创建prometheus-operator

```shell
[root@rain-kubernetes-1 kube-prometheus]# cd prometheus-operator/

[root@rain-kubernetes-1 prometheus-operator]# kubectl apply -f bundle.yaml
clusterrolebinding.rbac.authorization.k8s.io/prometheus-operator created
clusterrole.rbac.authorization.k8s.io/prometheus-operator created
deployment.apps/prometheus-operator created
serviceaccount/prometheus-operator created
service/prometheus-operator created

[root@rain-kubernetes-1 prometheus-operator]# kubectl get pod | grep prome
prometheus-operator-99dccdc56-565pn   1/1     Running            0          4m49s
[root@rain-kubernetes-1 prometheus-operator]# kubectl get deployment | grep prome
prometheus-operator   1/1     1            1           4m56s
[root@rain-kubernetes-1 prometheus-operator]# kubectl get service | grep prome
prometheus-operator   ClusterIP      None            <none>                  8080/TCP       5m1s
```

Prometheus-operator部署了一个deployment，deployment里包含有一个pod，并且还创建了一个service

而prometheus-operator pod起来之后，就创建了一个APIService，以及5个CRD

```shell
[root@rain-kubernetes-1 prometheus-operator]# kubectl get APIService | grep monitor
v1.monitoring.coreos.com               Local     True        7m30s

level=info ts=2019-11-28T03:13:15.33330737Z caller=operator.go:1863 component=prometheusoperator msg="CRD created" crd=Prometheus
level=info ts=2019-11-28T03:13:15.412107674Z caller=operator.go:634 component=alertmanageroperator msg="CRD created" crd=Alertmanager
level=info ts=2019-11-28T03:13:15.460722373Z caller=operator.go:1863 component=prometheusoperator msg="CRD created" crd=ServiceMonitor
level=info ts=2019-11-28T03:13:15.556857575Z caller=operator.go:1863 component=prometheusoperator msg="CRD created" crd=PodMonitor
level=info ts=2019-11-28T03:13:15.665412281Z caller=operator.go:1863 component=prometheusoperator msg="CRD created" crd=PrometheusRule
```

而创建的serviceaccount使得prometheus-operator有权限对`monitoring.coreos.com`这个apiGroup里的CRD进行所有操作

## 部署prometheus server

部署prometheus server使用prometheus crd就可以，prometheus crd的实现实际上是一个statefulset

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
---
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
spec:
  serviceMonitorSelector:
    matchLabels:
      team: node-exporter
  serviceAccountName: prometheus
  resources:
    requests:
      memory: 400Mi
```

```shell
[root@rain-kubernetes-1 prometheus-operator]# kubectl apply -f prometheus.yml
serviceaccount/prometheus created
prometheus.monitoring.coreos.com/prometheus created

[root@rain-kubernetes-1 prometheus-operator]# kubectl get statefulset
NAME                    READY   AGE
prometheus-prometheus   1/1     59s

[root@rain-kubernetes-1 prometheus-operator]# kubectl get pod prometheus-prometheus-0 -o wide
NAME                      READY   STATUS    RESTARTS   AGE   IP           NODE                NOMINATED NODE   READINESS GATES
prometheus-prometheus-0   3/3     Running   1          30m   10.168.2.5   rain-kubernetes-3   <none>           <none>
```

prometheus-prometheus创建了一个pod，包含了三个container: [prometheus prometheus-config-reloader rules-configmap-reloader]

## 部署pod以及service

首先通过daemonset在每个node上部署node-exporter，然后创建一个service，使得这些node-exporter作为该service的endpoint

```yaml
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      name: node-exporter
  template:
    metadata:
      labels:
        name: node-exporter
    spec:
      containers:
      - name: node-exporter
        image: prom/node-exporter
        ports:
        - name: metrics
          containerPort: 9100
          
---
apiVersion: v1
kind: Service
metadata:
  name: node-exporter-service
  labels:
    name: node-exporter-service
spec:
  selector:
    name: node-exporter
  ports:
  - name: metrics
    port: 9100
```

```shell
[root@rain-kubernetes-1 prometheus-operator]# kubectl get ds
NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
node-exporter   3         3         3       3            3           <none>          56m

[root@rain-kubernetes-1 prometheus-operator]# kubectl get service | grep node-exporter
node-exporter-service   ClusterIP      10.111.69.190   <none>                  9100/TCP       2m33s

[root@rain-kubernetes-1 prometheus-operator]# kubectl get pod | grep node-exporter
node-exporter-5lxqm                   1/1     Running            0          56m
node-exporter-hq97n                   1/1     Running            0          56m
node-exporter-lsdm4                   1/1     Running            0          56m

[root@rain-kubernetes-1 prometheus-operator]# kubectl get endpoints | grep node-exporter
node-exporter-service   10.168.0.6:9100,10.168.1.2:9100,10.168.2.6:9100         8m46s
```

## 部署ServiceMonitor

创建一个ServiceMonitor告诉prometheus server我们要监控刚刚创建的node-exporter-service背后的一组pod

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: node-exporter-servicemonitor
  labels:
    team: node-exporter
spec:
  selector:
    matchLabels:
      name: node-exporter-service
  endpoints:
  - port: metrics
```

```shell
[root@rain-kubernetes-1 prometheus-operator]# kubectl apply -f servicemonitor.yml
servicemonitor.monitoring.coreos.com/node-exporter-servicemonitor created
```

默认情况下`ServiceMonitor`和监控对象必须是在相同Namespace下的，如果要关联其他namespace下的

```yaml
spec:
  namespaceSelector:
    matchNames:
    - target_ns_name
```

如果希望关联任意namespaces下的

```yaml
spec:
  namespaceSelector:
    any: true
```

servicemonitor可以通过两个方式来定义target：

1. 通过selector来根据labels选取对应service的endpoints，它描述了Prometheus Server的Target列表（那些serivce背后的endpoints），Operator 会监听这个资源的变化来动态的更新Prometheus Server的Scrape targets并让prometheus server去reload配置。

   > Tips：定义servicemonitor的时候，定义了一个labels  `team: node-exporter`，这个要跟prometheus server，在这边是部署promethus的时候的   `matchLabels:  team: node-exporter` 对应起来。（调了好一会儿，差点要放弃了，555）意思是，prometheus server会从labels满足`team: node-exporter`的servicemonitor去获取监控target。如果想一个prometheus server监控所有的则`spec.serviceMonitorSelector: {}`为空即可

2. 上面的定义是依赖于service&pod的，那对于一个没有pod or service的外部服务，要怎么监控呢？比如etcd

创建endpoints，手工输入etcd的ip

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: etcd-metrics
  labels:
    team: etcd-exporter
subsets:
  - addresses:
    - ip: 10.225.28.220
    - ip: 10.225.28.171
    - ip: 10.225.28.227
    ports:
    - name: metrics
      port: 2381
      protocol: TCP
```

然后创建service关联endpoints，注意这里通过metadata里的name关联（endpoints跟service的metadata的name要一致）

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    team: etcd-exporter
  name: etcd-metrics
spec:
  clusterIP: None
  ports:
  - name: metrics
    port: 2381
    targetPort: 2381
  selector: null
```

然后同样创建一个servicemonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    team: etcd-expoter
  name: etcd-servicemonitor
spec:
  endpoints:
  - port: metrics
  selector:
    matchLabels:
      team: etcd-exporter
```

然而，失败了，原因是需要tls认证

![image-20191128161706039](/assets/images/image-20191128161706039.png)

修改一下servicemonitor，先手工检查一下（也整了半天。。）

* 需要healthcheck的key和cert
* ip需要跟etcd --listen-metrics-urls 匹配 （在我的环境里，--listen-metrics-urls=http://127.0.0.1，所以怎么都不行），要把这个配置改一下，/etc/kubernetes/manifests/etcd.yaml，`--listen-metrics-urls=http://127.0.0.1:2381,https://10.225.28.220:2381`，然后下面的curl才可以work了

```shell
curl --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/healthcheck-client.crt --key /etc/kubernetes/pki/etcd/healthcheck-client.key --insecure https://10.225.28.220:2381/metrics
```

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    team: etcd-expoter
  name: etcd-servicemonitor
spec:
  endpoints:
  - port: metrics
    scheme: https
    tlsConfig:
      caFile: /etc/prometheus/secrets/etcd-client-certs/ca.crt
      certFile: /etc/prometheus/secrets/etcd-client-certs/client.crt
      keyFile: /etc/prometheus/secrets/etcd-client-certs/client.key
      serverName: etcd-cluster
      insecureSkipVerify: yes
  selector:
    matchLabels:
      team: etcd-exporter
```

需要创建一个secret

```shell
[root@rain-kubernetes-1 prometheus-operator]# kubectl create secret generic etcd-client-certs --from-file=ca.crt=/etc/kubernetes/pki/etcd/ca.crt --from-file=client.crt=/etc/kubernetes/pki/etcd/healthcheck-client.crt --from-file=client.key=/etc/kubernetes/pki/etcd/healthcheck-client.key
```

然后把这个secret加载到promethes server上，修改prometheus的Yaml文件

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
---
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
spec:
  serviceMonitorSelector: {}
  serviceAccountName: prometheus
  secrets:
  - etcd-client-certs
  resources:
    requests:
      memory: 400Mi
```

etcd-client-certs加载后，会生成在容器内的`/etc/prometheus/secrets/etcd-client-certs/`目录下，然后在servicemonitor里，定义一下tlsconfig

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    team: etcd-expoter
  name: etcd-servicemonitor
spec:
  endpoints:
  - port: metrics
    scheme: https
    tlsConfig:
      caFile: /etc/prometheus/secrets/etcd-client-certs/ca.crt
      certFile: /etc/prometheus/secrets/etcd-client-certs/client.crt
      keyFile: /etc/prometheus/secrets/etcd-client-certs/client.key
      serverName: etcd-cluster
      insecureSkipVerify: yes
  selector:
    matchLabels:
      team: etcd-exporter
```

![image-20191128174617839](/assets/images/image-20191128174617839.png)

只有第二个work，是因为我只改了一台上面的etcd配置

##  创建prometheus service

创建prometheus service以便我们可以方便方便通过web查看metrics

```yaml
apiVersion: v1
kind: Service
metadata:
  name: prometheus
spec:
  type: LoadBalancer
  selector:
    prometheus: prometheus
  ports:
  - name: metrics
    port: 9091
    protocol: TCP
    targetPort: 9090
```

```shell
[root@rain-kubernetes-1 prometheus-operator]# kubectl get service prometheus
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
prometheus   LoadBalancer   10.100.96.144   10.225.28.229   9091:32412/TCP   54m
```

![image-20191128174751872](/assets/images/image-20191128174751872.png)