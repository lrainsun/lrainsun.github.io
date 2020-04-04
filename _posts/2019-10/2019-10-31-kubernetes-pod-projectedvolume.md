---
layout: post
title:  "Kubernetes Pod Projected Volume"
date:   2019-10-31 21:30:00 +0800
categories: Kubernetes
tags: Kubernetes-Pod
excerpt: Learn about Kubernetes Pod Projected Volume
mathjax: true
typora-root-url: ../
---

Pod是可以挂载volume的，这个volume为Pod里不同的container所共享。一般volume都是为了存放container里的数据，或者进行container和host之间的数据交换。

插一嘴，kubernetes的volume是attach到pod上的，volume的生命周期跟pod是一致的，跟container没啥关系，container被销毁或者重建，volume是不受影响的。每个container可以定义volumeMounts来使用volume。

但除此之外，Pod里还有一种特殊的volume，称作为Projected（投射） Volume（kubernetes v1.11后支持），它们的作用是为container提供预先定义好的数据。

Kubernetes支持的projected volumes一共有四种:

1. Secret 
2. ConfigMap
3. Downward API （一直被我看成download了，今天才算看明白是downward）
4. ServiceAccountToken （其实是Secret的一种）

# Secret

顾名思义，Secret就是保存的加密的数据，这个加密的数据是保存在etcd中的，可以通过挂载volume的方式，访问到这些加密数据。

假设我们现在有一份加密数据，里面的信息包括用户名和密码：

```
[root@rain-kubernetes-1 rain]# cat 15-credential.yaml
username: rain
password: password
```

我们先要把它保存到etcd中，也就是用Secret对象的方式交给kubernetes保存：

```
[root@rain-kubernetes-1 rain]# kubectl create secret generic credential --from-file=./15-credential.yaml
secret/credential created

[root@rain-kubernetes-1 rain]# kubectl get secrets
NAME                  TYPE                                  DATA   AGE
credential            Opaque                                1      50s

[root@rain-kubernetes-1 rain]# kubectl get secrets credential -o yaml
apiVersion: v1
data:
  15-credential.yaml: dXNlcm5hbWU6IHJhaW4KcGFzc3dvcmQ6IHBhc3N3b3JkCg==
kind: Secret
metadata:
  creationTimestamp: "2019-10-31T02:18:12Z"
  name: credential
  namespace: default
  resourceVersion: "954075"
  selfLink: /api/v1/namespaces/default/secrets/credential
  uid: 5ac0080f-7e7c-4d3d-9d91-ee5a916a2681
type: Opaque
```

Secret的data部分存放的key-value就是我们的credential

> Tips：实际data部分存放的数据仅仅是经过了base64转码，并没有做真正的加密，生产环境中是需要通过加密插件来进行加密，增强数据安全性的

现在创建一个pod，把secret作为projected volume挂载进去：

```
[root@rain-kubernetes-1 rain]# cat 15-test-secret-projected-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-secrect-projected-volume
spec:
  containers:
  - name: test-secret-projected-volume
    image: busybox
    args:
    - sleep
    - "86400"
    volumeMounts:
    - name: cred
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: cred
    projected:
      sources:
      - secret:
          name: credential
        
[root@rain-kubernetes-1 rain]# kubectl apply -f 15-test-secret-projected-volume.yaml
pod/test-secrect-projected-volume created

[root@rain-kubernetes-1 rain]# kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
test-secrect-projected-volume   1/1     Running   0          16s
```

pod running之后可以进到container里查看：

```
[root@rain-kubernetes-1 rain]# kubectl exec -it test-secrect-projected-volume -- /bin/sh
/ # ls /projected-volume/
15-credential.yaml

/ # cat /projected-volume/15-credential.yaml
username: rain
password: password
```

保存在etcd里的用户密码信息已经以文件形式出现在了container volume目录里

一旦对应的etcd里数据被更新，这些volume里文件的内容也会被更新，kubelet会定时维护这些volume（数据的更新会有一定的延时）

# CongfigMap

configmap跟secret是很类似的，区别在于，secret里保存的是需要加密的数据，configmap里是不需要加密的，通常是应用所需要的一些配置信息

把一份配置文件保存成configmap：

```
[root@rain-kubernetes-1 rain]# cat 15-config.yaml
name: rain
team: ocp

[root@rain-kubernetes-1 rain]# kubectl create configmap configfile --from-file=15-config.yaml
configmap/configfile created

[root@rain-kubernetes-1 rain]# kubectl get configmap
NAME         DATA   AGE
configfile   1      7s

[root@rain-kubernetes-1 rain]# kubectl get configmap configfile -o yaml
apiVersion: v1
data:
  15-config.yaml: |
    name: rain
    team: ocp
kind: ConfigMap
metadata:
  creationTimestamp: "2019-10-31T02:53:45Z"
  name: configfile
  namespace: default
  resourceVersion: "957179"
  selfLink: /api/v1/namespaces/default/configmaps/configfile
  uid: 5183298a-c35f-48fb-be0b-534ce37a676d
```

然后同样创建一个pod把这个configmap挂在成projected volume

```
[root@rain-kubernetes-1 rain]# cat 15-test-configmap-projected-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-secrect-projected-volume
spec:
  containers:
  - name: test-secret-projected-volume
    image: busybox
    args:
    - sleep
    - "86400"
    volumeMounts:
    - name: config
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: config
    projected:
      sources:
      - configmap:
          name: configfile
          
[root@rain-kubernetes-1 rain]# kubectl apply -f 15-test-configmap-projected-volume.yaml
pod/test-configmap-projected-volume created

[root@rain-kubernetes-1 rain]# kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
test-configmap-projected-volume   1/1     Running   0          10s
```

依然可以进到container里查看一下配置文件有没有在指定目录下了：

```
[root@rain-kubernetes-1 rain]# kubectl exec -it test-configmap-projected-volume -- /bin/sh
/ # ls /projected-volume/
15-config.yaml

/ # cat /projected-volume/15-config.yaml
name: rain
team: ocp
```

# Downward API

作用是，让Pod里的容器能够直接获取到这个Pod API对象本身的信息。这个信息一定是pod里的容器启动之前就能够确定下来的信息。如果要容器运行后才有的信息，downward api是做不到的。

```
[root@rain-kubernetes-1 rain]# cat 15-downward-api.yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: test-downwardapi-volume
  labels:
    zone: china
    cluster: cluster1
    rack: rack-22
spec:
  containers:
  - name: client-container
    image: busybox
    command: ["sh", "-c"]
    args:
    - while true; do
        if [[ -e /etc/podinfo/labels ]]; then
          echo -en '\n\n'; cat /etc/podinfo/labels; fi;
        sleep 5;
      done;
    volumeMounts:
    - name: podinfo
      mountPath: /etc/podinfo
      readOnly: false
  volumes:
  - name: podinfo
    projected:
      sources:
      - downwardAPI:
          items:
          - path: "labels"
            fieldRef:
              fieldPath: metadata.labels
              
[root@rain-kubernetes-1 rain]# kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
test-downwardapi-volume           1/1     Running   0          8s

[root@rain-kubernetes-1 rain]# kubectl logs test-downwardapi-volume
cluster="cluster1"
rack="rack-22"
zone="china"
```

这个例子通过downward API获取到了pod的一些信息

# Service Account

每个Pod在创建的时候，kubernetes都会默默的给它的spec.volumes加上一个volume，就是ServiceAccountToken，其实是Secret的一种。

如果不指定，在pod的spec.volumes里会定义一个默认的token

```
  - name: default-token-6xqqx
    secret:
      defaultMode: 420
      secretName: default-token-6xqqx
```

而在pod中的每个container都会把这个token mount到指定目录

```
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-6xqqx
      readOnly: true
```

```
/ # ls /var/run/secrets/kubernetes.io/serviceaccount
ca.crt     namespace  token
```

这下面是一些证书和授权文件，使用这些，在pod内部就可以call kubernetes API了。