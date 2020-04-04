---
layout: post
title:  "Kubernetes Pod网络"
date:   2019-11-19 19:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Network
excerpt: Kubernetes Pod网络
mathjax: true
typora-root-url: ../
---

我用的网络插件是flannel vxlan模式，所以先以这个为例子学习一下

# Pod内容器间通信

上次学习到kubernetes网络通信包括有:

1. 同一Pod内的容器间通信
2. Pod之间的通信

最早Pod的时候就学习过：

1. Pod内的容器其实是在同一个namespace里的
2. 在Pod内的容器启动之前，会有一个infra容器先被启动，用来hold住network namespace

```shell
[root@rain-kubernetes-1 rain]# kubectl get pod | grep rain-test
rain-test                           2/2     Running            0          12s
```

比如我创建了一个测试的Pod rain-test，其中运行着两个容器，用docker ps看一下

```shell
[root@rain-kubernetes-2 ~]# docker ps | grep rain-test
b33a26790d2e        busybox                                                          "sh -c 'echo The app…"   2 minutes ago       Up 2 minutes                            k8s_busybox2_rain-test_default_8b9fcf67-23ca-4654-aedc-3600ef91235f_0
e16c597d9523        busybox                                                          "sh -c 'echo The app…"   2 minutes ago       Up 2 minutes                            k8s_busybox1_rain-test_default_8b9fcf67-23ca-4654-aedc-3600ef91235f_0
4046bf3a022d        k8s.gcr.io/pause:3.1                                             "/pause"                 2 minutes ago       Up 2 minutes                            k8s_POD_rain-test_default_8b9fcf67-23ca-4654-aedc-3600ef91235f_0
```

实际有三个容器，其中一个就是我们所说的infra容器。看下这三个容器的进程号

```shell
[root@rain-kubernetes-2 ~]# docker inspect --format '\{\{ .State.Pid \}\}' k8s_busybox2_rain-test_default_8b9fcf67-23ca-4654-aedc-3600ef91235f_0
831
[root@rain-kubernetes-2 ~]# docker inspect --format '\{\{ .State.Pid \}\}' k8s_busybox1_rain-test_default_8b9fcf67-23ca-4654-aedc-3600ef91235f_0
759
[root@rain-kubernetes-2 ~]# docker inspect --format '\{\{ .State.Pid \}\}' k8s_POD_rain-test_default_8b9fcf67-23ca-4654-aedc-3600ef91235f_0
618
```

再看下他们进程对应的Network namespace，其实都是一个

```shell
[root@rain-kubernetes-2 ~]# ls -al /proc/831/ns | grep net
lrwxrwxrwx. 1 root root 0 Nov 18 14:15 net -> net:[4026533356]
[root@rain-kubernetes-2 ~]# ls -al /proc/759/ns | grep net
lrwxrwxrwx. 1 root root 0 Nov 18 14:15 net -> net:[4026533356]
[root@rain-kubernetes-2 ~]# ls -al /proc/618/ns | grep net
lrwxrwxrwx. 1 root root 0 Nov 18 14:11 net -> net:[4026533356]
```

```shell
[root@rain-kubernetes-2 ~]# docker exec -u root -it k8s_busybox2_rain-test_default_8b9fcf67-23ca-4654-aedc-3600ef91235f_0 ifconfig
eth0      Link encap:Ethernet  HWaddr A6:54:C8:EB:AA:BE
          inet addr:10.168.1.199  Bcast:0.0.0.0  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:6 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:480 (480.0 B)  TX bytes:42 (42.0 B)
          
[root@rain-kubernetes-2 ~]# docker exec -u root -it k8s_busybox1_rain-test_default_8b9fcf67-23ca-4654-aedc-3600ef91235f_0 ifconfig
eth0      Link encap:Ethernet  HWaddr A6:54:C8:EB:AA:BE
          inet addr:10.168.1.199  Bcast:0.0.0.0  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:6 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:480 (480.0 B)  TX bytes:42 (42.0 B)
```

正因为他们在同一个Network namespace，所以他们看到的网络视图都是一致的，那么自然而然就可以像本地进程一样进行通信，如果同一个pod内的两个容器监听了同一端口，那就会有冲突的。

# 同一主机Pod间通信

那位于同一个节点的，多个Pod间的通信是怎么做到的呢？

kubernetes会在每台宿主机上创建一个单独的网桥，默认的设备名是 cni0。网桥（Bridge）是能够起到虚拟交换机作用的网络设备，它是一个工作在数据链路层（Data Link）的设备，主要功能是根据 MAC 地址学习来将数据包转发到网桥的不同端口（Port）上。

```shell
[root@rain-kubernetes-1 rain]# ifconfig cni0
cni0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.168.0.1  netmask 255.255.255.0  broadcast 0.0.0.0
        inet6 fe80::9ce9:a6ff:fe1f:bd47  prefixlen 64  scopeid 0x20<link>
        ether 9e:e9:a6:1f:bd:47  txqueuelen 1000  (Ethernet)
        RX packets 5959377  bytes 441626893 (421.1 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 7129287  bytes 2030106468 (1.8 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

有了这样的一个bridge之后，下面要做的，就是把每个容器连接到cni0这个网桥上，那么自然而然就通了咯。

连接的方式是通过Veth Pair：

> Veth Pair 设备的特点是：它被创建出来后，总是以两张虚拟网卡（Veth Peer）的形式成对出现的。并且，从其中一个“网卡”发出的数据包，可以直接出现在与它对应的另一张“网卡”上，哪怕这两个“网卡”在不同的 Network Namespace 里。
> 这就使得 Veth Pair 常常被用作连接不同 Network Namespace 的“网线”。

网络插件flannel需要负责把Veth Pair的一端连接到cni0上，另一端连接到容器里

比如rain-kubernetes-1上有两个pod：

```shell
[root@rain-kubernetes-1 rain]# kubectl get pod -o wide| grep rain-kubernetes-1
coffee-8c8ff9b4f-ftnfh              1/1     Running            0          7d17h   10.168.0.12    rain-kubernetes-1   <none>           <none>
tea-658d56f6cc-5pkjc                1/1     Running            0          7d17h   10.168.0.11    rain-kubernetes-1   <none>           <none>
```

进到coffee的container里面

```shell
[root@rain-kubernetes-1 rain]# docker exec -u root -it k8s_coffee_coffee-8c8ff9b4f-ftnfh_default_5f634f4d-f8e0-4d90-b4a9-85b06b64bad6_0 sh
/ # ifconfig
eth0      Link encap:Ethernet  HWaddr CE:02:AE:31:B1:46
          inet addr:10.168.0.12  Bcast:0.0.0.0  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:922 errors:0 dropped:0 overruns:0 frame:0
          TX packets:33 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:41724 (40.7 KiB)  TX bytes:3760 (3.6 KiB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
          
/ # cat /sys/class/net/eth0/iflink
16

/ # ip link show eth0
3: eth0@if16: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue state UP
    link/ether ce:02:ae:31:b1:46 brd ff:ff:ff:ff:ff:ff

/ # route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         10.168.0.1      0.0.0.0         UG    0      0        0 eth0
10.168.0.0      *               255.255.255.0   U     0      0        0 eth0
10.168.0.0      10.168.0.1      255.255.0.0     UG    0      0        0 eth0
```

这个容器里面有一个eth0的网卡，这就是Veth Pair设备在容器里的一端

通过route可以看到路由表，eth0是容器的默认路由设备。所有10.168.0.0/24，10.168.0.0/16的请求，也会被交给eth0来处理

而这个Veth Pair设备的另一端，就在宿主机上。

在宿主机上可以找到与刚刚容器内对应的Veth Pair的另一端，两种方式都可以

```shell
# 遍历/sys/claas/net下面的全部目录查看子目录ifindex的值和容器里面查出来的iflink值相当的veth名字
[root@rain-kubernetes-1 rain]# grep "16" -rn /sys/class/net/veth*/ifindex
/sys/class/net/veth912977f8/ifindex:1:16

# 容器内ip link show eth0 命令，然后可以看到3: eth0@if16，其中3是eth0接口的index, 16是和他pair的veth的index
[root@rain-kubernetes-1 rain]# ip link show | grep 16
16: veth912977f8@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master cni0 state UP mode DEFAULT
```

所以coffee这个容器的Veth Pair的另一端就在宿主机的veth912977f8上

```shell
[root@rain-kubernetes-1 rain]# ifconfig veth912977f8
veth912977f8: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet6 fe80::d0d9:dbff:fe9b:a545  prefixlen 64  scopeid 0x20<link>
        ether d2:d9:db:9b:a5:45  txqueuelen 0  (Ethernet)
        RX packets 35  bytes 3885 (3.7 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 924  bytes 41933 (40.9 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

同理，tea容器Veth Pair的对端在宿主机的veth96c9a6b1上

```shell
[root@rain-kubernetes-1 rain]# docker exec -u root -it k8s_tea_tea-658d56f6cc-5pkjc_default_0c4500a9-aa9e-405c-920b-afa1d34d03f9_0 sh
/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 0A:01:7E:DC:64:CC
          inet addr:10.168.0.11  Bcast:0.0.0.0  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:911 errors:0 dropped:0 overruns:0 frame:0
          TX packets:23 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:40631 (39.6 KiB)  TX bytes:2376 (2.3 KiB)
```

```shell
[root@rain-kubernetes-1 rain]# brctl show cni0
bridge name     bridge id               STP enabled     interfaces
cni0            8000.9ee9a61fbd47       no              veth912977f8
                                                        veth96c9a6b1
                                                        vetha1f27b10
                                                        vetha3dcbb37
                                                        vethc2d7e1a2
                                                        vethd0f1afa2
```

veth96c9a6b1跟veth912977f8都被连接到了cni0网桥上，所以tea跟coffee这两个容器自然就互通了。原理是，比如coffee要访问tea

1. 在coffee容器里要ping 10.168.0.11

2. 根据coffee容器里的路由，会匹配 ```10.168.0.0      *               255.255.255.0   U     0      0        0 eth0``` 这条路由，这是一条直连规则，即：凡是匹配到这条规则的 IP 包，应该经过本机的 eth0 网卡，通过二层网络直接发往目的主机。

3. 通过二层到tea，就需要有10.168.0.11这个IP地址对应的MAC地址，所以coffee会通过eth0网卡先发送一个ARP广播，来通过IP地址查找到对端的MAC地址

4. 而由于coffee容器里的eth0网卡，是一个Veth Pair的一端，另一端在宿主机上，而且连接到宿主机的cni0上

   > 一旦一张虚拟网卡被“插”在网桥上，它就会变成该网桥的“从设备”。从设备会被“剥夺”调
   > 用网络协议栈处理数据包的资格，从而“降级”成为网桥上的一个端口。而这个端口唯一的作
   > 用，就是接收流入的数据包，然后把这些数据包的“生杀大权”（比如转发或者丢弃），全部交
   > 给对应的网桥。

5. 所以收到ARP请求后，其实cni0会扮演二层交换机的角色，把ARP广播转发到其他也同样连接到cni0的虚拟网卡上

6. 因为tea的eth0的另一端也连在cni0上，所以tea就会收到这个ARP请求，然后把自己的MAC地址回复给coffee

7. coffee知道了tea的MAC地址，就可以通过eth0网卡把数据包发出去了

8. 同样数据包发出去，会出现在宿主机的veth912977f8上（如上，veth912977f8的网络协议栈资格被"剥夺"），然后数据包直接流入到cni0网桥

   > 对宿主机来说，cni0 网桥就是一个普通的网卡

9. cni0网桥上流入的数据包，就会通过宿主机的网络协议栈进行处理

   ```
   [root@rain-kubernetes-1 rain]# route | grep 10.168.0.0
   10.168.0.0      0.0.0.0         255.255.255.0   U     0      0        0 cni0
   ```

10. 根据如上规则，数据包会由cni0再转发（forward）出去

11. cni0网桥根据数据包的目的 MAC 地址，在它的 CAM 表（即交换机通过 MAC地址学习维护的端口和 MAC 地址的对应表）里查到对应的端口（Port）为：veth96c9a6b1，然后把数据包发往这个端口

12. 这是veth pair的一端，另一端就在tea里，所以最终coffee就发数据发给了tea

---

在默认情况下，被限制在 NetworkNamespace 里的容器进程，实际上是通过 Veth Pair 设备 + 宿主机网桥的方式，实现了跟同其他容器的数据交换。

盗图，意思是一样的，docker0 -> cni0 懒得画图了

![image-20191119171212492](/assets/images/image-20191119171212492.png)

宿主机访问本宿主机容器的IP：

![image-20191119171440134](/assets/images/image-20191119171440134.png)

容器访问另外一个宿主机：

![image-20191119171120411](/assets/images/image-20191119171120411.png)

# 跨主机Pod间通信

![image-20191119171939878](/assets/images/image-20191119171939878.png)

跨主通信主要靠的是这个Overlay Network（我们需要在已有的宿主机网络上，再通过软件构建一
个覆盖在已有宿主机网络之上的、可以把所有容器连通在一起的虚拟网络），容器的对端Veth Pair都会被连接到Overley Network上实现联通性。

Overlay Network 本身，可以由每台宿主机上的一个“特殊网桥”共同组成。比如，当Node 1 上的 Container 1 要访问 Node 2 上的 Container 3 的时候，Node 1 上的“特殊网桥”在收到数据包之后，能够通过某种方式，把数据包发送到正确的宿主机，比如 Node 2上。而 Node 2 上的“特殊网桥”在收到数据包后，也能够通过某种方式，把数据包转发给正确的容器，比如 Container 3。

> VXLAN，即 Virtual Extensible LAN（虚拟可扩展局域网），是 Linux 内核本身就支持的一种网络虚似化技术。所以说，VXLAN 可以完全在内核态实现上述封装和解封装的工作，从而通过“隧道”机制，构建出覆盖网络（Overlay Network）。
> VXLAN 的覆盖网络的设计思想是：在现有的三层网络之上，“覆盖”一层虚拟的、由内核 VXLAN模块负责维护的二层网络，使得连接在这个 VXLAN 二层网络上的“主机”（虚拟机或者容器都可以）之间，可以像在同一个局域网（LAN）里那样自由通信。当然，实际上，这些“主机”可能分布在不同的宿主机上，甚至是分布在不同的物理机房里。
> 而为了能够在二层网络上打通“隧道”，VXLAN 会在宿主机上设置一个特殊的网络设备作为“隧道”的两端。这个设备就叫作 VTEP，即：VXLAN Tunnel End Point（虚拟隧道端点）。
> 而 VTEP 设备的作用，其实跟前面的 flanneld 进程非常相似。只不过，它进行封装和解封装的对象，是二层数据帧（Ethernet frame）；而且这个工作的执行流程，全部是在内核里完成的（因为VXLAN 本身就是Linux 内核中的一个模块）。

![image-20191119173031571](/assets/images/image-20191119173031571.png)

每台宿主机上有个名叫 flannel.1 的设备，就是 VXLAN 所需的 VTEP 设备，它既有 IP地址，也有 MAC 地址。

```shell
[root@rain-kubernetes-1 rain]# kubectl get pod -o wide| grep rain-kubernetes-1 | grep coffee
coffee-8c8ff9b4f-ftnfh              1/1     Running            0          7d19h   10.168.0.12    rain-kubernetes-1   <none>           <none>
[root@rain-kubernetes-1 rain]# kubectl get pod -o wide| grep rain-kubernetes-2 | grep tea
tea-658d56f6cc-vq82j                1/1     Running            0          7d19h   10.168.1.32    rain-kubernetes-2   <none>           <none>
```

rain-kubernetes-1上的coffee要与rain-kubernetes-2上的tea通信（10.168.0.12 -> 10.168.1.32）

1. 跟上面一样，coffee发出消息后，会先出现在cni0网桥

2. 然后会被路由到flannel.1进行处理，这里是"隧道"的入口，把这个 IP 包称为“原始 IP 包”

   ```shell
   [root@rain-kubernetes-1 rain]# route | grep 10.168.1
   10.168.1.0      10.168.1.0      255.255.255.0   UG    0      0        0 flannel.1
   ```

3. 为了能够将“原始 IP 包”封装并且发送到正确的宿主机，VXLAN 就需要找到这条“隧道”的出口，即：目的宿主机的 VTEP 设备。这个设备的信息，正是每台宿主机上的 flanneld 进程负责维护的

4. 比如rain-kubernetes-2启动并加入flannel网络后，在其他所有node上，flannel都会添加如上的路由规则。凡是发往 10.168.1.0/24 网段的 IP 包，都需要经过 flannel.1 设备发出，并且，它最后被发往的网关地址是：10.168.1.0

5. 10.168.1.0正是rain-kubernetes-2上的vtep设备（也就是 flannel.1 设备）的 IP 地址

   ```shell
   [root@rain-kubernetes-2 ~]# ifconfig flannel.1
   flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
           inet 10.168.1.0  netmask 255.255.255.255  broadcast 0.0.0.0
           inet6 fe80::b4b2:a8ff:fe5f:2c32  prefixlen 64  scopeid 0x20<link>
           ether b6:b2:a8:5f:2c:32  txqueuelen 0  (Ethernet)
           RX packets 1328123  bytes 109299325 (104.2 MiB)
           RX errors 0  dropped 0  overruns 0  frame 0
           TX packets 1328039  bytes 224042853 (213.6 MiB)
           TX errors 0  dropped 15 overruns 0  carrier 0  collisions 0
   ```

6. rain-kubernetes-1 flannel.1称为“源 VTEP 设备“，rain-kubernetes-2 flannel.1称为“目的 VTEP 设备“
7. 这些 VTEP 设备之间，就需要想办法组成一个虚拟的二层网络，即：通过二层数据帧进行通信

   >  “源 VTEP 设备”收到“原始 IP 包”后，就要想办法把“原始 IP 包”加上一个目的 MAC 地址，封装成一个二层数据帧，然后发送给“目的 VTEP 设备”（当然，这么做还是因为这个 IP 包的目的地址不是本机）

8. “目的 VTEP 设备”的 IP地址是10.168.1.0，那么MAC 地址是什么？这是ARP做的事情。而这个ARP记录并不是发送ARP请求获得来的，而是，rain-kubernetes-2在启动的时候，除了在其他主机添加路由外，还添加了ARP记录（当然也是flannel进程干的）

   ```shell
   [root@rain-kubernetes-1 rain]# ip neigh show dev flannel.1
   10.168.2.0 lladdr 6a:0f:f2:87:94:25 PERMANENT
   10.168.1.0 lladdr b6:b2:a8:5f:2c:32 PERMANENT   # rain-kubernetes-2的MAC地址
   ```

9. 有了这个“目的 VTEP 设备”的 MAC 地址，Linux 内核就可以开始二层封包工作了。这个二层帧的格式是这样的。Linux 内核会把“目的 VTEP 设备”的 MAC 地址，填写在图中的 Inner Ethernet Header 字段，得到一个二层数据帧。封包过程只是加一个二层头，不会改变“原始 IP 包”的内容。所以图中的Inner IP Header 字段，依然是tea的 IP 地址，即 10.168.1.32。VTEP 设备的 MAC 地址，对于宿主机网络来说并没有什么实际意义。所以封装出来的这个数据帧，并不能在宿主机二层网络里传输。可以称为“内部数据帧”（Inner Ethernet Frame）

   ![image-20191119183845861](/assets/images/image-20191119183845861.png)

   > 目的VTEP设备MAC地址 -> b6:b2:a8:5f:2c:32
   >
   > 目的容器的IP地址 -> 10.168.1.32

10. 因为内部数据帧没办法在宿主机二层网络里传输，所以要进一步封装，成为一个宿主机里的普通数据帧，称为外部数据帧。Linux 内核会在“内部数据帧”前面，加上一个特殊的 VXLAN头，用来表示这个外部数据帧里面实际上是一个 VXLAN 要使用的数据帧。

    >  这个 VXLAN 头里有一个重要的标志叫作VNI，它是 VTEP 设备识别某个数据帧是不是应该归自己处理的重要标识。而在 Flannel 中，VNI 的默认值是 1，这也是为何，宿主机上的 VTEP 设备都叫作 flannel.1 的原因，这里的“1”，其实就是 VNI 的值。

11. Linux 内核会把这个数据帧封装进一个 UDP 包里发出去。从宿主机来看，是从一个宿主机的flannel.1设备向另一台宿主机设备的flannel.1设备发起的UDP连接

12. 可是，rain-kubernetes-1上的flannel.1只知道rain-kubernetes-2上的flannel.1设备的MAC地址，却并不知道rain-kubernetes-2的宿主机的地址，那UDP包要发给哪台宿主机呢？

13. flannel.1 设备实际上要扮演一个“网桥”的角色，在二层网络进行 UDP 包的转发。而在 Linux 内核里面，“网桥”设备进行转发的依据，来自于一个叫作 FDB（Forwarding Database）的转发数据库。所以我们要发给10.225.28.171这台主机，就是rain-kubernetes-2

    ```shell
    [root@rain-kubernetes-1 rain]# bridge fdb show flannel.1 | grep b6:b2:a8:5f:2c:32
    b6:b2:a8:5f:2c:32 dev flannel.1 dst 10.225.28.171 self permanent
    
    [root@rain-kubernetes-2 ocp]# ifconfig eth0
    eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
            inet 10.225.28.171  netmask 255.255.255.0  broadcast 10.225.28.255
            inet6 fe80::f816:3eff:fe43:13e  prefixlen 64  scopeid 0x20<link>
            ether fa:16:3e:43:01:3e  txqueuelen 1000  (Ethernet)
            RX packets 73247023  bytes 10643903336 (9.9 GiB)
            RX errors 0  dropped 14  overruns 0  frame 0
            TX packets 17687011  bytes 1836551697 (1.7 GiB)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
    ```

14. 接下来的流程，就是一个正常的、宿主机网络上的封包工作。UDP 包是一个四层数据包，所以 Linux 内核会在它前面加上一个 IP 头，即原理图中的Outer IP Header，组成一个 IP 包。并且，在这个 IP 头里，会填上前面通过 FDB 查询出来的目的主机的 IP 地址，即 rain-kubernetes-2 的 IP 地址10.225.28.171。然后，Linux 内核再在这个 IP 包前面加上二层数据帧头，即原理图中的 Outer Ethernet Header，并把 rain-kubernetes-2 的 MAC 地址（fa:16:3e:43:01:3e）填进去。这个 MAC 地址本身，是 rain-kubernetes-1 的 ARP 表要学习的内容，无需 Flannel 维护。这时候，我们封装出来的“外部数据帧”的格式，如下所示

    ![image-20191119185433849](/assets/images/image-20191119185433849.png)

    >  目的主机MAC地址 -> fa:16:3e:43:01:3e
    >
    > 目的主机IP地址 -> 10.225.28.171
    >
    > 目的VTEP设备MAC地址 -> b6:b2:a8:5f:2c:32
    >
    > 目的容器的IP地址 -> 10.168.1.32

15. 接下来，rain-kubernetes-1 上的 flannel.1 设备就可以把这个数据帧从 rain-kubernetes-1的 eth0 网卡发出去。显然，这个帧会经过宿主机网络来到 rain-kubernetes-2的 eth0 网卡。
16. rain-kubernetes-2 内核网络栈会发现这个数据帧里有 VXLAN Header，并且 VNI=1。所以 Linux内核会对它进行拆包，拿到里面的内部数据帧，然后根据 VNI 的值，把它交给 rain-kubernetes-2 上的flannel.1 设备。
17. rain-kubernetes-2 上的flannel.1 设备则会进一步拆包，取出“原始 IP 包”。最终，IP 包就进入到了tea容器的 Network Namespace里。

