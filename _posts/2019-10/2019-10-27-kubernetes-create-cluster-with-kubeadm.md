---
layout: post
title:  "kubeadm部署kubernetes cluster"
date:   2019-10-27 23:30:00 +0800
categories: Kubernetes
tags: Kubernetes-Env
excerpt: 用kubeadm部署kubernetes cluster
mathjax: true
---

# 我的环境配置

三台vm（1个master，2个Worker）

- CentOS 7.4
- 4 CPU core + 4G Mem + 80G Disk
  <br/>

# 安装RunTime

我用的容器运行时是Docker

```
yum -y install docker-ce

systemctl enable docker.service
systemctl start docker.service
```

<br/>

# 安装 kubeadm, kubelet 和 kubectl

添加kubernetes repo

```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF
```

> Tips: kubelet 的版本不可以超过 API server 的版本

禁用SELinux，容器才能访问宿主的文件系统，进而能够正常使用 Pod 网络

```
# 将 SELinux 设置为 permissive 模式(将其禁用)
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

systemctl enable kubelet && systemctl start kubelet
```

加载br_netfilter模块，并设置net.bridge.bridge-nf-call-iptables为1，防止iptables 被绕过导致网络请求被错误的路由

```
modprobe br_netfilter

​```bash
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
​```
```

<br/>

# 初始化Cluster (master节点)

pod-network-cidr是Pod网络的CIDR，由网络插件来管理，要跟网络插件的配置相匹配，这个地方如果不配pod-network-cidr，后面在安装网络插件的时候会报错

> Error registering network: failed to acquire lease: node "rain-kubernetes-1" pod cidr not assigned

如果用flannel，那么默认CIDR是**`10.244.0.0/16`**，这个地方可以直接用**`10.244.0.0/16`**，可是我手欠，非改了IP，那后面配flannel的时候也要相应改一下

```
kubeadm init --pod-network-cidr 10.168.0.0/16
```

kubeadm init大概要跑几分钟，跑完之后，cluster就建好了，最后会输出如下信息：

> Your Kubernetes control-plane has initialized successfully!
>
> To start using your cluster, you need to run the following as a regular user:
>
>  mkdir -p $HOME/.kube
>
>  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
>
>  sudo chown $(id -u):$(id -g) $HOME/.kube/config
>
> You should now deploy a pod network to the cluster.
>
> Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
>
>  https://kubernetes.io/docs/concepts/cluster-administration/addons/
>
> Then you can join any number of worker nodes by running the following on each as root:
>
> kubeadm join 10.225.28.220:6443 --token si5jqp.y0nc9lkmc5bhkjtj \
>
>   --discovery-token-ca-cert-hash sha256:cebdd4c20f793d107736aa81900e203d133fe5bb3ed9c3d3bc897b525d1d9d73

最后的这一行kubeadm join命令要保存好，worker node在加入cluster的时候需要执行这个命令

为了方便运行kubectl，可以把kubeconfig拷贝到如下指定目录：

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

然后，可以查看一下当前的状态

```
[root@rain-kubernetes-1 ~]# kubectl get nodes
NAME STATUS ROLES AGE VERSION
rain-kubernetes-1 NotReady master 7m15s v1.16.2
```

discribe一下：

> Ready False Wed, 23 Oct 2019 09:00:01 +0000 Wed, 23 Oct 2019 08:51:51 +0000 KubeletNotReady runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady messa

是因为还没安装网络插件，所以node没有ready

<br/>

# 安装网络插件

网络插件我选了flannel

```
wget https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml
```

因为前面pod网络填了**`--pod-network-cidr 10.168.0.0/16`**，所以这个地方要多一步，yml文件里配置要改成一致：

```
  net-conf.json: |
    {
      "Network": "10.168.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
```

然后，apply一下

```
kubectl apply -f kube-flannel.yml
```

等网络插件起来后，再去看node，状态已经是ready了

这个时候就可以去添加worker node了

<br/>

# 添加worker node

这一部超简单的，在两台worker节点上，执行刚过保存下来的kubeadm join命令就好啦

```
kubeadm join 10.225.28.220:6443 --token si5jqp.y0nc9lkmc5bhkjtj \
  --discovery-token-ca-cert-hash sha256:cebdd4c20f793d107736aa81900e203d133fe5bb3ed9c3d3bc897b525d1d9d73
```

再get node一下看：

```
[root@rain-kubernetes-1 ~]# kubectl get nodes
NAME                STATUS   ROLES    AGE    VERSION
rain-kubernetes-1   Ready    master   4d2h   v1.16.2
rain-kubernetes-2   Ready    <none>   4d2h   v1.16.2
rain-kubernetes-3   Ready    <none>   4d2h   v1.16.2
```

<br/>

# Master Taint

master默认会绑定一个key为node-role.kubernetes.io/master的taint，为了使得master也可以run pod，我们可以删除master上的taint

```
[root@rain-kubernetes-1 ~]# kubectl taint nodes --all node-role.kubernetes.io/master-
node/rain-kubernetes-1 untainted
```

<br/>

# 安装etcdctl

为了方便学习，安装一下ETCD Client

```
wget https://github.com/etcd-io/etcd/releases/download/v3.4.1/etcd-v3.4.1-linux-amd64.tar.gz
tar xzvf etcd-v3.4.1-linux-amd64.tar.gz
cd etcd-v3.4.1-linux-amd64
ls
cp etcdctl /usr/local/bin/
```

在master上可以直接查询ETCD数据库

```
[root@rain-kubernetes-1 ~]# ETCDCTL_API=3 etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key endpoint status -w table
+----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|    ENDPOINT    |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 127.0.0.1:2379 | b9b7f43639f04584 |  3.3.15 |  1.6 MB |      true |      false |         2 |     623786 |                  0 |        |
+----------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

