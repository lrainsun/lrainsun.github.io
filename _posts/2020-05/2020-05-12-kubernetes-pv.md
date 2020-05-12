---
layout: post
title:  "Kubernetes创建volume"
date:   2020-05-12 23:00:00 +0800
categories: Kubernetes
tags: Kubernetes-Volume
excerpt: Kubernetes创建volume
mathjax: true
typora-root-url: ../
---

# Kubernetes存储系统nfs server的部署

我们通过helm来部署nfs server

## 安装helm

```shell
wget https://get.helm.sh/helm-v3.2.1-linux-amd64.tar.gz
tar -zxvf helm-v3.2.1-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
```

# 安装nfs-server-provisioner

## nfs-server-provisioner & nfs-client-provisioner

nfs-server-provisioner顾名思义是一个nfs server的Provisioner。。然后就是在pod里面起一个nfs

而nfs-client-provisioner使用外面的nfs

嗯，这个问题我纠结半天。。

## helm安装nfs-server-provisioner

```shell
[root@ocp-dev-003 helm]# helm repo update

[root@ocp-dev-003 helm]# kubectl create namespace airflow
namespace/airflow created
[root@ocp-dev-003 helm]# helm install nfs-server-provisioner stable/nfs-server-provisioner -n airflow -f helm-nfs-values.yaml
```

```shell
[root@ocp-dev-003 ~]# helm list -n airflow
NAME                  	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART                       	APP VERSION
nfs-server-provisioner	airflow  	1       	2020-05-12 09:09:01.049535065 +0000 UTC	deployed	nfs-server-provisioner-1.0.0	2.3.0

[root@ocp-dev-003 airflow-yaml]# kubectl get sc -n airflow
NAME   PROVISIONER                            RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs    cluster.local/nfs-server-provisioner   Delete          Immediate           true                   10m
```

PV 这个对象的创建，是由运维人员完成的。但是，在大规模的生产环境里，这其实是一个非常麻烦的工作。这是因为，一个大规模的 Kubernetes 集群里很可能有成千上万个 PVC，这就意味着运维人员必须得事先创建出成千上万个 PV。更麻烦的是，随着新的 PVC 不断被提交，运维人员就不得不继续添加新的、能满足条件的 PV，否则新的 Pod 就会因为 PVC 绑定不到 PV 而失败。在实际操作中，这几乎没办法靠人工做到。

storageclass的目的是Dynamic Provisioning PV，它的作用，其实就是创建 PV 的模板。

而 StorageClass 对象的作用，其实就是创建 PV 的模板

# 创建pvc

我们只需要创建需要的pvc就可以，pv会自动被创建

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: airflow-dags
  namespace: airflow
spec:
  accessModes:
  - ReadOnlyMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: nfs
  volumeMode: Filesystem
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: airflow-logs
  namespace: airflow
spec:
  accessModes:
  - ReadOnlyMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: nfs
  volumeMode: Filesystem
```

```shell
[root@ocp-dev-003 airflow-yaml]# kubectl get pvc -n airflow
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
airflow-dags   Bound    pvc-aed00b2c-f8f1-40a5-8f06-4c269166bb52   1Gi        ROX            nfs            33s
airflow-logs   Bound    pvc-ef090ca3-becc-4053-8693-1d454b39f82a   1Gi        ROX            nfs            33s
[root@ocp-dev-003 airflow-yaml]# kubectl get pv -n airflow
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   REASON   AGE
pvc-aed00b2c-f8f1-40a5-8f06-4c269166bb52   1Gi        ROX            Delete           Bound    airflow/airflow-dags   nfs                     35s
pvc-ef090ca3-becc-4053-8693-1d454b39f82a   1Gi        ROX            Delete           Bound    airflow/airflow-logs   nfs                     34s
```

describe一下创建出来的pv

```yaml
  nfs:
    path: /export/pvc-aed00b2c-f8f1-40a5-8f06-4c269166bb52
    server: 10.99.141.169
  persistentVolumeReclaimPolicy: Delete
  storageClassName: nfs
  volumeMode: Filesystem
status:
  phase: Bound
```

这个nfs server就是nfs server provision提供出来的，ip就是暴露出来的service的ip

```shell
[root@ocp-dev-003 airflow-yaml]# kubectl get pods -n airflow
NAME                       READY   STATUS    RESTARTS   AGE
nfs-server-provisioner-0   1/1     Running   0          16m

[root@ocp-dev-003 airflow-yaml]# kubectl get service -n airflow
NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                                                                     AGE
nfs-server-provisioner   ClusterIP   10.99.141.169   <none>        2049/TCP,2049/UDP,32803/TCP,32803/UDP,20048/TCP,20048/UDP,875/TCP,875/UDP,111/TCP,111/UDP,662/TCP,662/UDP   12m
```

这里并没有做持久化，也就是说，只要nfs-server-provisioner-0这个pod存在，volume就会存在

# References

[1] [https://helm.sh/docs/intro/quickstart/](https://helm.sh/docs/intro/quickstart/)

[2] [https://helm.sh/docs/intro/install/](https://helm.sh/docs/intro/install/)

[3] [https://github.com/helm/charts/tree/master/stable/nfs-server-provisioner](https://github.com/helm/charts/tree/master/stable/nfs-server-provisioner)