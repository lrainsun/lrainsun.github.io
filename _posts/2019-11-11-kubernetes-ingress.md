---
layout: post
title:  "Kubernetes Ingress"
date:   2019-11-11 23:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Service
excerpt: Kubernetes Service和Pod的DNS
mathjax: true
typora-root-url: ../
---

# Ingress简介

kubernetes有两种内建的云端负载均衡机制：

* Service资源实现的是"TCP负载均衡器"
  * Service资源和Pod资源的IP地址仅能用于集群网络内部的通信，所有的网络流量都无法穿透边界路由器以实现集群内外通信
  * 无论是iptables还是ipvs模型的service资源都配置于linux内核中netfilter之上进行四层调度，是一种类型更为通用的调度器，支持调度http，mysql等应用层服务
  * 尽管可以为service使用nodeport或loadbalaner类型通过节点引入外部流量，但依然是四层流量转发，可用的负载均衡器也为传输层负载均衡机制
  * 无法做到类似卸载https中ssl会话等一类操作，也不支持基于url的请求调度机制
* Ingress资源实现的是“HTTP(S)负载均衡器”
  * 是应用层负载机制的一种，支持根据环境做出更好的调度决策
  * 提供了诸如可自定义url映射和tls卸载等功能
  * Ingress是一组基于DNS名称（host）或URL路径把请求转发至指定的service资源的规则，用于将集群外部的请求流量转发至集群内部完成服务发布
  * `Ingresss`是`k8s`集群中的一个`API`资源对象，扮演边缘路由器(edge router)的角色，也可以理解为**集群防火墙**、**集群网关**，我们可以**自定义路由规则**来转发、管理、暴露服务(一组pod)，非常灵活，生产环境建议使用这种方式。
  * Ingress 的功能其实很容易理解：所谓 Ingress，就是 Service 的“Service”。

# 部署Ingress Controller

Ingress资源定义了转发规则，而根据这些规则的匹配机制路由请求流量，让这些规则发挥作用的则是Ingress Controller，所以我们需要部署Ingress Controller

```shell
[root@rain-kubernetes-1 rain]# kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
namespace/ingress-nginx created
configmap/nginx-configuration created
configmap/tcp-services created
configmap/udp-services created
serviceaccount/nginx-ingress-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-role created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-role-nisa-binding created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-clusterrole-nisa-binding created
deployment.apps/nginx-ingress-controller created

[root@rain-kubernetes-1 rain]# kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS    RESTARTS   AGE
nginx-ingress-controller-568867bf56-hzhsj   1/1     Running   0          174m
```

与其他类型的控制器不同，其他类型的控制器通常作为`kube-controller-manager`二进制文件的一部分运行，Ingress controller是单独部署的。

一个 Nginx Ingress Controller会监视来自于API Server的Ingress对象状态，并以其规则生成相应的应用程序专有格式的配置文件，并通过重载或者重启守护进程而使得新配置生效。对于Nginx来说，Ingress规则就需要转换为Nginx的配置信息，跟踪Ingress资源并实时生成配置规则。其实就是一个可以根据 Ingress对象和被代理后端 Service 的变化，来自动进行更新的 Nginx 负载均衡器。

Ingress控制器可以由任何具有反向代理功能的服务程序实现，Ingress控制器自身也是运行于集群中的Pod资源对象

## 接入外部流量

可是，同样运行为Pod资源的Ingress控制器又怎么接入外部的请求流量呢？？？一般有两种方式：

1. 使用专用的Service对象为Ingress控制器接入外部流量

![image-20191110230414166](/assets/images/image-20191110230414166.png)

以Deployment控制器管理Ingress控制器的Pod资源，并通过nodeport或loadbalancer类型的service对象为其接入集群外部的请求流量，这就意味着，定义一个ingress控制器的时候，必须在其前端定义一个专用的service对象

2. 以hostport或者hostnetwork的方式为Ingress控制器接入外部流量

![image-20191110230815672](/assets/images/image-20191110230815672.png)

借助于DaemonSet控制器，将Ingress控制器的Pod资源各自以单一实例的方式运行与集群的所有或者部分工作节点上，并配置这类Pod对象以hostport或者hostnetwork的方式在当前节点接入外部流量

## 部署Ingress Controller Service

我们采用第一种方式试试看，所以刚刚我们创建了Ingress Controller Pod，现在还需要一个Service

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
    - name: https
      port: 443
      targetPort: 443
      protocol: TCP
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
```

这个 Service 的唯一工作，就是将所有携带 ingress-nginx 标签的 Pod 的 80 和433 端口暴露出去

```shell
[root@rain-kubernetes-1 rain]# kubectl get service -n ingress-nginx
NAME                       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller   NodePort   10.100.42.151   <none>        80:31547/TCP,443:30795/TCP   29m
```

这样就可以通过\<nodeip\>:\<nodeport\>来使用了

# 创建Ingress资源

Ingress资源是基于http虚拟主机或者url的转发规则，Ingress Spec中的字段是定义Ingress资源的核心组成部分，主要嵌套如下三个字段：

* rules
  * 用于定义当前Ingress资源的转发规则列表
  * 如果没有定义rules，或者没有匹配到任何规则，所有流量都会转发到由backend定义的默认后端
  * 由一系列配置Ingress资源的host规则组成，这些host（host目前不支持使用IP地址，也不支持后跟:PORT格式的端口号，如果host留空表示通配所有的主机名）规则用于将一个主机的某个url路径映射至相关的后端service对象
* backend
  * 默认的后端用于服务那些没有匹配到任何规则的请求，用于让负载均衡器指定一个全局默认的后端
  * 定义Ingress资源时，至少应该定义backend或者rules两者之一
  * 有两个内嵌字段ServiceName & ServicePort，分别用于指定流量转发的后端目标service资源的名称和端口
* tls
  * tls配置，目前仅支持通过默认端口443提供服务
  * 仅在定义TLS主机的转发规则时才需要定义此类对象
  * 有两个内嵌字段，hosts & secretName

# Ingress资源类型

## 单Service资源型Ingress

像Service中的NodePort一样，Ingress也可以用来暴露服务，只需要为Ingress指定“default backend"即可，我们试一下之前获取hostname的例子通过Ingress来暴露服务

```yaml
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: hostnames-ingress
spec:
  backend:
    serviceName: hostnames
    servicePort: 80
```

其实backend定义的就是一个service，所以首先hostnames这个service得存在

```shell
[root@rain-kubernetes-1 rain]# kubectl get service
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP             PORT(S)   AGE
hostnames       ClusterIP      10.104.83.227   <none>                  80/TCP    2d5h
```

apply之后，可以看到Ingress资源被创建，并且根据定义的backend找到了三个pod

```shell
[root@rain-kubernetes-1 rain]# kubectl get ingress hostnames-ingress -o wide
NAME                HOSTS   ADDRESS          PORTS   AGE
hostnames-ingress   *       10.100.132.187   80      28h

[root@rain-kubernetes-1 rain]# kubectl describe ingress hostnames-ingress
Name:             hostnames-ingress
Namespace:        default
Address:
Default backend:  hostnames:80 (10.168.0.10:9376,10.168.1.12:9376,10.168.2.21:9376)
Rules:
  Host  Path  Backends
  ----  ----  --------
  *     *     hostnames:80 (10.168.0.10:9376,10.168.1.12:9376,10.168.2.21:9376)
```

现在就可以用我们刚刚生成的ingress controller service对外暴露的nodeport或者cluster ip或Ingress Address来访问了

```shell
[root@rain-kubernetes-1 rain]# kubectl get service -n ingress-nginx
NAME                       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller   NodePort   10.100.42.151   <none>        80:31547/TCP,443:30795/TCP   29m

[root@rain-kubernetes-1 rain]# curl 10.225.28.220:31547
hostnames-5db5477647-8ggkg

[root@rain-kubernetes-1 rain]# curl 10.100.42.151
hostnames-5db5477647-8ggkg

[root@rain-kubernetes-1 rain]# curl 10.100.132.187
hostnames-5db5477647-zlg5h
```

这里我们还没有部署外部负载均衡器，所以external-ip是空的

# 基于url路径进行流量转发

假如我现在有这样一个站点：https://cafe.example.com。其中，https://cafe.example.com/coffee，对应的是“咖啡点餐系统”。而，https://cafe.example.com/tea，对应的则是“茶水点餐系统”。这两个系统，分别由名
叫 coffee 和 tea 这样两个 Deployment 来提供服务。
那么现在，我如何能使用 Kubernetes 的 Ingress 来创建一个统一的负载均衡器，从而实现当用户访问不同的域名时，能够访问到不同的 Deployment 呢？

创建一个Ingress对象：

```yaml
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cafe-ingress
spec:
  rules:
  - host: cafe.example.com
    http:
      paths:
      - path: /tea
        backend:
          serviceName: tea-srv
          servicePort: 80
      - path: /coffee
        backend:
          serviceName: coffee-srv
          servicePort: 80
```

rules字段定义了IngressRule：

* IngressRule 的 Key，就叫做：host。它必须是一个标准的域名格式（Fully Qualified Domain Name）的字符串，而不能是 IP 地址。
* 而 host 字段定义的值，就是这个 Ingress 的入口。，当用户访问cafe.example.com 的时候，实际上访问到的是这个 Ingress 对象。这样，Kubernetes 就能使用 IngressRule 来对你的请求进行下一步转发。
* IngressRule 规则的定义，则依赖于 path 字段，这里的每一个path 都对应一个后端 Service。

https://github.com/resouer/kubernetes-ingress/tree/master/examples/complete-example

创建tea-srv & coffee-srv

```shell
[root@rain-kubernetes-1 rain]# kubectl describe ingress cafe-ingress
Name:             cafe-ingress
Namespace:        default
Address:          10.100.132.187
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host              Path  Backends
  ----              ----  --------
  cafe.example.com
                    /tea      tea-svc:80 (10.168.0.11:80,10.168.1.32:80,10.168.2.25:80)
                    /coffee   coffee-svc:80 (10.168.0.12:80,10.168.1.33:80)
```

尝试访问

```shell
[root@rain-kubernetes-1 rain]# curl --resolve cafe.example.com:31547:10.225.28.220 http://cafe.example.com:31547/tea
Server address: 10.168.1.32:80
Server name: tea-658d56f6cc-vq82j
Date: 11/Nov/2019:15:09:52 +0000
URI: /tea
Request ID: b56c242420a77e55248087502841656a

[root@rain-kubernetes-1 rain]# curl --resolve cafe.example.com:31547:10.225.28.220 http://cafe.example.com:31547/coffee
Server address: 10.168.0.12:80
Server name: coffee-8c8ff9b4f-ftnfh
Date: 11/Nov/2019:15:09:56 +0000
URI: /coffee
Request ID: f3bbbc2cf384e72043f50a17fa1e567b
```

访问https://cafe.example.com:443/coffee时，应该是 coffee 这个Deployment 负责响应我的请求

访问https://cafe.example.com:433/tea的时候，则应该是 tea 这个Deployment 负责响应我的请求

那在这里，Service起到了什么作用呢？

* 我们定义Ingress规则的时候，需要由一个Service资源对象来辅助识别相关的所有Pod对象
* ingress-nginx控制器可根据规则的定义，直接将请求流量调度至Pod（在这里就是coffee & tea Pod），而无需由Service对象API的再次转发

所以，采用Ingress进行流量分发时，Ingress的转发机制会绕过Service资源，从而省去了由kube-proxy实现的端口代理开销

# 基于主机名称的虚拟主机

刚刚的例子改造一下

```yaml
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cafe-ingress-host
spec:
  rules:
  - host: tea.com
    http:
      paths:
      - path: /
        backend:
          serviceName: tea-svc
          servicePort: 80
  - host: coffee.com
    http:
      paths:
      - path: /
        backend:
          serviceName: coffee-svc
          servicePort: 80
```

把coffee & tea分到两个域名上去

```shell
[root@rain-kubernetes-1 rain]# kubectl get ingress  -o wide
cafe-ingress-host   tea.com,coffee.com   10.100.132.187   80        31m

[root@rain-kubernetes-1 rain]# kubectl describe ingress cafe-ingress-host
Name:             cafe-ingress-host
Namespace:        default
Address:          10.100.132.187
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host        Path  Backends
  ----        ----  --------
  tea.com
              /   tea-svc:80 (10.168.0.11:80,10.168.1.32:80,10.168.2.25:80)
  coffee.com
              /   coffee-svc:80 (10.168.0.12:80,10.168.1.33:80)
```

尝试访问一下：

```shell
[root@rain-kubernetes-1 rain]# curl --resolve tea.com:31547:10.225.28.220 http://tea.com:31547
Server address: 10.168.1.32:80
Server name: tea-658d56f6cc-vq82j
Date: 11/Nov/2019:15:25:12 +0000
URI: /
Request ID: 6d64f229d90f6ad6fe505abcd1c19f8d

[root@rain-kubernetes-1 rain]# curl --resolve coffee.com:31547:10.225.28.220 http://coffee.com:31547
Server address: 10.168.1.33:80
Server name: coffee-8c8ff9b4f-865pf
Date: 11/Nov/2019:15:25:21 +0000
URI: /
Request ID: 940bf7fdba70e2ec7715b00c49014249
```

# TLS类型的Ingress资源

改造前面的例子，变成tls类型

我们需要创建 Ingress 所需的 SSL 证书（tls.crt）和密钥（tls.key），这些信息都是通过Secret 对象定义好的

```shell
[root@rain-kubernetes-1 cert]# kubectl get secret cafe-secret
NAME          TYPE     DATA   AGE
cafe-secret   Opaque   2      55s
[root@rain-kubernetes-1 cert]# kubectl describe secret cafe-secret
Name:         cafe-secret
Namespace:    default
Labels:       <none>
Annotations:
Type:         Opaque

Data
====
tls.crt:  1164 bytes
tls.key:  1675 bytes
```

把tls配置加到之前的Ingress里

```yaml
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cafe-ingress
spec:
  tls:
  - hosts:
    - cafe.example.com
    secretName: cafe-secret
  rules:
  - host: cafe.example.com
    http:
      paths:
      - path: /tea
        backend:
          serviceName: tea-svc
          servicePort: 80
      - path: /coffee
        backend:
          serviceName: coffee-svc
          servicePort: 80
```

试着访问一下

```shell
[root@rain-kubernetes-1 rain]# kubectl get ingress  -o wide
NAME                HOSTS                ADDRESS          PORTS     AGE
cafe-ingress        cafe.example.com     10.100.132.187   80, 443   123m

[root@rain-kubernetes-1 rain]# kubectl get service -n ingress-nginx
NAME                       TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller   NodePort   10.100.42.151    <none>        80:31547/TCP,443:30795/TCP   24h

[root@rain-kubernetes-1 rain]# curl --resolve cafe.example.com:30795:10.225.28.220 https://cafe.example.com:30795/tea --insecure
Server address: 10.168.1.32:80
Server name: tea-658d56f6cc-vq82j
Date: 11/Nov/2019:15:51:53 +0000
URI: /tea
Request ID: 556d52cc760fb40201ff63aa9229dcf6

[root@rain-kubernetes-1 rain]# curl --resolve cafe.example.com:30795:10.225.28.220 https://cafe.example.com:30795/coffee --insecure
Server address: 10.168.1.33:80
Server name: coffee-8c8ff9b4f-865pf
Date: 11/Nov/2019:15:51:56 +0000
URI: /coffee
Request ID: 8fecc6e1a30dd265759df724582f0af7
```



