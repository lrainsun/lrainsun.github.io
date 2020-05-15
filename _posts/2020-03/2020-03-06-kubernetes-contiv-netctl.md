---
layout: post
title:  "VPP运维工具contiv-netctl"
date:   2020-03-06 23:00:00 +0800
categories: Kubernetes
tags: Kubernetes-Network
excerpt: VPP运维工具contiv-netctl
mathjax: true
typora-root-url: ../
---

# contiv-netctl

![Contiv-netctl](/../assets/images/contiv-netctl-arch.png)

contiv-netctl是一个可以用来查询contiv-VPP的工具，在安装contiv-VPP CNI插件的时候它默认被安装在k8s的master节点上

```shell
/ # /contiv-netctl help
Usage:
  netctl [command]

Available Commands:
  help        Help about any command
  ipam        Shows IPAM information for specified node.
  nodes       Shows vswitch information for all nodes in the running Contiv cluster.
  pods        Display network information for pods connected to VPP on the given node. If node is omitted, pod data for all nodes is shown.
  vppcli      Execute the specified VPP debug CLI command on the specified node.
  vppdump     Print anything to the screen

Flags:
      --etcd-config string          path to etcd.conf config file
  -h, --help                        help for netctl
      --http-client-config string   path to http.client.conf config file

Use "netctl [command] --help" for more information about a command.
```

* 获取node和pod信息：需要连接到contiv-etcd（etcd中存了contiv pod和node的基本信息，以及他们分布在哪里）
* 获取contiv-vswitch信息，支持vpp cli：需要跟contiv-vpp agent（每个contiv-vswitch都有一个contiv-vpp agent）进行通信

# 使用contiv-netctl

登录到master节点

```shell
[root@oke-prod-ci22sj-a-master-0 ~]# cat /usr/local/bin/contiv-netctl
#!/bin/sh
kubectl get pods -n kube-system | \
  grep contiv-crd | \
  cut -d " " -f 1 | \
  xargs -I{} kubectl exec -n kube-system {} \
  /contiv-netctl "$@"
```

其实是进到了contiv-crd里执行了contiv-netctl命令

不然的话，需要通过这样的方式提供两个配置文件：

> Since contiv-netctl connects to vswitches via REST and uses Contiv-ETCD it requires corresponding config files. They can be defined using command line arguments `--etcd-config` and `--http-client-config` or environment variables `ETCD_CONFIG`, `HTTP_CLIENT_CONFIG`.

```shell
/ # /contiv-netctl nodes
INFO[0000] Connected to Etcd (took 1.428629ms)           endpoints="[10.241.89.68:32379]" loc="etcd/bytes_broker_impl.go(60)" logger=defaultLogger
ID  NODE-NAME                    VPP-IP         HOST-IP        START-TIME                STATE  BUILD-VERSION  BUILD-DATE
1   oke-prod-ci22sj-a-master-0   10.241.89.60   10.241.89.68   Fri Mar  6 01:57:18 2020  OK     v3.2.0         Tue Jul  2 11:45:00 2019
3   oke-prod-ci22sj-a-master-1   10.241.89.71   10.241.89.67   Fri Mar  6 01:48:52 2020  OK     v3.2.0         Tue Jul  2 11:45:00 2019
2   oke-prod-ci22sj-a-master-2   10.241.89.63   10.241.89.56   Fri Mar  6 01:49:22 2020  OK     v3.2.0         Tue Jul  2 11:45:00 2019
4   oke-prod-ci22sj-a-worker-1   10.241.89.57   10.241.89.54   Fri Mar  6 01:50:12 2020  OK     v3.2.0         Tue Jul  2 11:45:00 2019
```

其中vpp-ip是host上用来跨pod通讯的vxlan tunnel的endpoint的IP

```shell
/ # /contiv-netctl pods oke-prod-ci22sj-a-worker-26
INFO[0000] Connected to Etcd (took 1.25428ms)            endpoints="[10.241.89.68:32379]" loc="etcd/bytes_broker_impl.go(60)" logger=defaultLogger
POD-NAME                                                       NAMESPACE                   POD-IP         IF-IDX  IF-NAME
cdpprod01sjc-cdp-backend-7b4b84d4f7-db74s                      cdp                         10.1.29.57     0       N/A
cdpprod01sjc-cdp-web-9df75bb89-78f9j                           cdp                         10.1.29.55     0       N/A
contiv-vswitch-27v9t                                           kube-system                 10.241.90.212
```

vppcli可以帮你在特定node上执行vppctl的命令

比如，在node上

```shell
vpp# show nat44 addresses
NAT44 pool addresses:
10.241.89.63
  tenant VRF independent
  1 busy udp ports
  136 busy tcp ports
  0 busy icmp ports
NAT44 twice-nat pool addresses:
10.1.2.254
  tenant VRF independent
  0 busy udp ports
  0 busy tcp ports
  0 busy icmp ports
```

跟这样的效果是一样的

```shell
/ # /contiv-netctl vppcli oke-prod-ci22sj-a-master-2 show nat44 addresses
INFO[0000] Connected to Etcd (took 1.562061ms)           endpoints="[10.241.89.68:32379]" loc="etcd/bytes_broker_impl.go(60)" logger=defaultLogger
vppcli oke-prod-ci22sj-a-master-2 show nat44 addresses
"NAT44 pool addresses:
10.241.89.63
  tenant VRF independent
  1 busy udp ports
  136 busy tcp ports
  0 busy icmp ports
NAT44 twice-nat pool addresses:
10.1.2.254
  tenant VRF independent
  0 busy udp ports
  0 busy tcp ports
  0 busy icmp ports
```

vppdump是一些可以帮助调试的信息

```shell
/ # /contiv-netctl vppdump oke-prod-ci22sj-a-worker-26
INFO[0000] Connected to Etcd (took 1.409973ms)           endpoints="[10.241.89.68:32379]" loc="etcd/bytes_broker_impl.go(60)" logger=defaultLogger
Command usage: netctl vppdump oke-prod-ci22sj-a-worker-26 <cmd>:
cmd 0: bond-interface
cmd 1: dhcp-proxy
cmd 2: linux-interface-watcher
cmd 3: microservice
cmd 4: linux-interface
cmd 5: linux-arp
cmd 6: linux-ipt-rulechain-descriptor
cmd 7: linux-route
cmd 8: vpp-acl-to-interface
cmd 9: vpp-bd-interface
cmd 10: vpp-interface
cmd 11: vpp-acl
cmd 12: vpp-arp
cmd 13: vpp-bridge-domain
cmd 14: vpp-dhcp
cmd 15: vpp-interface-address
cmd 16: vpp-interface-link-state
cmd 17: vpp-interface-rx-mode
cmd 18: vpp-interface-rx-placement
cmd 19: vpp-interface-vrf
cmd 20: vpp-ip-scan-neighbor
cmd 21: vpp-l2-fib
cmd 22: vpp-nat44-address
cmd 23: vpp-nat44-dnat
cmd 24: vpp-nat44-global
cmd 25: vpp-nat44-interface
cmd 26: vpp-proxy-arp
cmd 27: vpp-proxy-arp-interface
cmd 28: vpp-punt-exception
cmd 29: vpp-punt-ipredirect
cmd 30: vpp-punt-to-host
cmd 31: vpp-route
cmd 32: vpp-sr-localsid
cmd 33: vpp-sr-policy
cmd 34: vpp-sr-steering
cmd 35: vpp-stn-rules
cmd 36: vpp-unnumbered-interface
cmd 37: vpp-vrf-table
cmd 38: vpp-xconnect
```

# Reference

[1] [https://github.com/contiv/vpp/blob/master/docs/operation/TOOLS.md](https://github.com/contiv/vpp/blob/master/docs/operation/TOOLS.md)

[2] [https://contivpp.io/blog/using-conti-vpp-netctl-blog/](https://contivpp.io/blog/using-conti-vpp-netctl-blog/)