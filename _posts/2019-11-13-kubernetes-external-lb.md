---
layout: post
title:  " Kubernetes External LB"
date:   2019-11-13 23:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Service
excerpt: Kubernetes External LoadBalancer
mathjax: true
typora-root-url: ../
---

# External LB

![image-20191110230414166](/assets/images/image-20191110230414166.png)

昨天看的Ingress里，还缺个externalLB，类型为 `LoadBalancer` 的服务在 Kubernetes 中并没有直接支持，其中Metallb是一个解决方案。

Metallb 会在 Kubernetes 内运行，监控服务对象的变化，一旦察觉有新的 `LoadBalancer` 服务运行，并且没有可申请的负载均衡器之后，就会完成两部分的工作：

1. 地址分配：用户需要在配置中提供一个地址池，Metallb 将会在其中选取地址分配给服务
2. 地址广播：根据不同配置，Metallb 会以二层（ARP for IPv4/NDP for IPv6）或者 BGP 的方式进行地址的广播

## Layer2 模式

Layer 2模式下，每个service会有集群中的一个node来负责。当服务客户端发起ARP解析的时候，对应的node会响应该ARP请求，之后，该service的流量都会指向该node（看上去该node上有多个地址）

Layer 2模式更为通用，不需要用户有额外的设备；但由于Layer 2模式使用ARP/ND，地址池分配需要跟客户端在同一子网，地址分配略为繁琐

Layer 2模式并不是真正的负载均衡，因为流量都会先经过1个node后，再通过kube-proxy转给多个end points。如果该node故障，MetalLB会迁移 IP到另一个node（the old leader’s lease times out after 10 seconds），并重新发送免费ARP告知客户端迁移

所以Layer 2模式有两个Limitaions:

1. 单节点瓶颈
2. failover可能会比较慢

## BGP 模式

BGP模式下，集群中所有node都会跟上联路由器建立BGP连接，并且会告知路由器应该如何转发service的流量

BGP模式是真正的LoadBalancer

# 部署metallb

当前最新版本是v0.8.3，我们通过yaml来部署

```shell
[root@rain-kubernetes-1 rain]# kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.3/manifests/metallb.yaml
namespace/metallb-system created
podsecuritypolicy.policy/speaker created
serviceaccount/controller created
serviceaccount/speaker created
clusterrole.rbac.authorization.k8s.io/metallb-system:controller created
clusterrole.rbac.authorization.k8s.io/metallb-system:speaker created
role.rbac.authorization.k8s.io/config-watcher created
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:controller created
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:speaker created
rolebinding.rbac.authorization.k8s.io/config-watcher created
daemonset.apps/speaker created
deployment.apps/controller created
```

主要有两个组件：

1. deployment.apps/controller，负责IP地址的分配，以及service和endpoint的。controller会监听service资源事件，一旦有LoadBalancer类型的service被创建时，就从ip地址池分配ip给service用。

   ```shell
   [root@rain-kubernetes-1 rain]# kubectl get deployment -n metallb-system
   NAME         READY   UP-TO-DATE   AVAILABLE   AGE
   controller   1/1     1            1           4m7s
   ```

2. daemonset.apps/speaker，负责保证service地址可达，例如Layer 2模式下，speaker会负责ARP请求应答

   ```shell
   [root@rain-kubernetes-1 rain]# kubectl get ds -n metallb-system
   NAME      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
   speaker   3         3         3       3            3           beta.kubernetes.io/os=linux   5m25s
   ```

部署完成后，还需要配置configMap，metallb-system/config，controller会读取该配置，并reload配置

## Layer2 模式部署

需要准备一个configMap，包含我们提供的vip地址

```yaml
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: config
  namespace: metallb-system
data:
  config: |
    address-pools:
    - name: rain-ip-pools
      protocol: layer2
      addresses:
      - 10.225.28.229-10.225.28.229
      - 10.225.28.239-10.225.28.239
```

# 创建loadbalaner Service

需要创建一个type是LoadBalancer的Service验证，偷懒还是改造一下之前的hostnames service

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: hostnames
spec:
  type: LoadBalancer
  selector:
    app: hostnames
  ports:
  - name: default
    protocol: TCP
    port: 80
    targetPort: 9376
```

部署完成后测试一下，有external-ip咯，通过这个external-ip就可以直接访问啦（因为我的kubernetes node是在openstack上的VM，所以这里还要多一步配allowed-address-pairs，这样从外面就可以访问这个external-ip了）

```shell
[root@rain-kubernetes-1 rain]# kubectl get service hostnames
NAME        TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
hostnames   LoadBalancer   10.104.83.227   10.225.28.229   80:30809/TCP   3d7h

[root@rain-kubernetes-1 rain]# curl 10.225.28.229
hostnames-5db5477647-8ggkg
```

# 工作原理

对于Layer2模式来说，IPv4用的是ARP

理论上来说，arp external-ip，得到的MAC地址应该是node的MAC地址，也就是会把流量引到node上去，but

```shell
[root@raincmp001 ~]# arping -I eth0 10.225.28.229
ARPING 10.225.28.229 from 10.225.12.151 eth0
Unicast reply from 10.225.28.229 [00:00:0C:07:AC:01]  0.940ms
Unicast reply from 10.225.28.229 [00:00:0C:07:AC:01]  35.369ms
Unicast reply from 10.225.28.229 [00:00:0C:07:AC:01]  0.863ms
^CSent 3 probes (1 broadcast(s))
Received 3 response(s)

[root@raincmp001 ~]# arping -I eth0 10.225.28.220
ARPING 10.225.28.220 from 10.225.12.151 eth0
Unicast reply from 10.225.28.220 [00:00:0C:07:AC:01]  1.295ms
Unicast reply from 10.225.28.220 [00:00:0C:07:AC:01]  0.873ms

[root@rain-kubernetes-1 rain]# ifconfig eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.225.28.220  netmask 255.255.255.0  broadcast 10.225.28.255
        inet6 fe80::f816:3eff:fe33:c9d8  prefixlen 64  scopeid 0x20<link>
        ether fa:16:3e:33:c9:d8  txqueuelen 1000  (Ethernet)
        RX packets 63746109  bytes 4941067683 (4.6 GiB)
        RX errors 0  dropped 13  overruns 0  frame 0
        TX packets 17865225  bytes 6912946353 (6.4 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

*00:00:0C:07:AC:01*并不是我node的Mac呀？

> This is normal behaviour in HSRP. When you ARP for the router virtual address, you get back the virtual MAC, for example 00:00:0c:07:ac:01 for group 1. So, for traffic to the router, a host will use the virtual MAC address as destination. But traffic coming back from the the router, (i.e. traffic that the router is forwarding to you) will have the router's built-in MAC address as source.

不纠结这个了，反正流量会被引到某一个kubernetes node上，而且只会被引导一个Node上:

比如我这个coffee-svc，

```shell
[root@rain-kubernetes-1 rain]# kubectl get service
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP             PORT(S)        AGE
coffee-svc      LoadBalancer   10.110.114.89   10.225.28.239           80:32015/TCP   40h

[root@rain-kubernetes-1 rain]# kubectl get pods -n metallb-system -o wide
NAME                          READY   STATUS    RESTARTS   AGE     IP              NODE                NOMINATED NODE   READINESS GATES
controller-65895b47d4-xktvl   1/1     Running   0          3h59m   10.168.1.35     rain-kubernetes-2   <none>           <none>
speaker-jt6f4                 1/1     Running   0          3h59m   10.225.28.171   rain-kubernetes-2   <none>           <none>
speaker-trqtt                 1/1     Running   0          3h59m   10.225.28.227   rain-kubernetes-3   <none>           <none>
speaker-zjqtf                 1/1     Running   0          3h59m   10.225.28.220   rain-kubernetes-1   <none>           <none>

[root@rain-kubernetes-1 rain]# kubectl logs speaker-jt6f4 -n metallb-system | grep 10.225.28.239
{"caller":"main.go:246","event":"serviceAnnounced","ip":"10.225.28.239","msg":"service has IP, announcing","pool":"rain-ip-pools","protocol":"layer2","service":"default/coffee-svc","ts":"2019-11-13T06:22:09.151613336Z"}
[root@rain-kubernetes-1 rain]# kubectl logs speaker-trqtt -n metallb-system | grep 10.225.28.239
[root@rain-kubernetes-1 rain]# kubectl logs speaker-zjqtf -n metallb-system | grep 10.225.28.239
```

只有rain-kubernetes-2上的speaker回复了arp响应，所以10.225.28.239的包都会先发到rain-kubernetes-2，然后再由rain-kubernetes-2进行转发。官网上说，metallb跟keepalived“看上去"有些类似呢，似乎都是一个host拥有了多个IP，只不过keepalived是通过vrrp协议来交换信息选leader的，而metallb则依赖于kubernetes来得知pod或者node是up还是down的。keepalived由于virtual router id是8bit的，最多只能是支持到255个load-balancer，而metallb就没有这个限制。但是metallb是依赖于kubernetes来代替标准网络协议，所以只能在kubernetes cluster内部提供load balancer的功能。

后面的故事，还是iptables来完成，看下node上的iptables规则：

```shell
[root@rain-kubernetes-1 rain]# iptables-save | grep loadbalancer | grep 10.225.28.229
-A KUBE-SERVICES -d 10.225.28.229/32 -p tcp -m comment --comment "default/hostnames:default loadbalancer IP" -m tcp --dport 80 -j KUBE-FW-ODX2UBAZM7RQWOIU

[root@rain-kubernetes-1 rain]# iptables-save | grep KUBE-FW-ODX2UBAZM7RQWOIU
:KUBE-FW-ODX2UBAZM7RQWOIU - [0:0]
-A KUBE-FW-ODX2UBAZM7RQWOIU -m comment --comment "default/hostnames:default loadbalancer IP" -j KUBE-MARK-MASQ
-A KUBE-FW-ODX2UBAZM7RQWOIU -m comment --comment "default/hostnames:default loadbalancer IP" -j KUBE-SVC-ODX2UBAZM7RQWOIU
-A KUBE-FW-ODX2UBAZM7RQWOIU -m comment --comment "default/hostnames:default loadbalancer IP" -j KUBE-MARK-DROP

-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000

-A KUBE-SVC-ODX2UBAZM7RQWOIU -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-UBFECYXO7WGMGUOV
-A KUBE-SVC-ODX2UBAZM7RQWOIU -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-SGM2NWFJJ2FS6TH7
-A KUBE-SVC-ODX2UBAZM7RQWOIU -j KUBE-SEP-7QEF2MJOSPQ44GT7
```

这些规则是不是很熟悉了，跟之前service的clusterip， nodeport的实现那几乎是一样一样的

# References

[1] [https://metallb.universe.tf/concepts/layer2/]( https://metallb.universe.tf/concepts/layer2/)

[2] [https://ieevee.com/tech/2019/06/30/metallb.html](https://ieevee.com/tech/2019/06/30/metallb.html)

