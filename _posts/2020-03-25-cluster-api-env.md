---
layout: post
title:  "cluster api环境搭建"
date:   2020-03-25 23:00:00 +0800
categories: Kubernetes
tags: Kubernetes-Env
excerpt: cluster api环境搭建
mathjax: true
typora-root-url: ../
---

# 安装cluster

我们用kind来安装一个极简cluster，前提工作时提前安装好docker和kubectl

```shell
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl

[root@rain-k8s cluster_api]# kubectl version
Client Version: version.Info{Major:"1", Minor:"17", GitVersion:"v1.17.4", GitCommit:"8d8aa39598534325ad77120c120a22b3a990b5ea", GitTreeState:"clean", BuildDate:"2020-03-12T21:03:42Z", GoVersion:"go1.13.8", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"13", GitVersion:"v1.13.4", GitCommit:"c27b913fddd1a6c480c229191a087698aa92f0b1", GitTreeState:"clean", BuildDate:"2019-03-20T16:54:39Z", GoVersion:"go1.11.5", Compiler:"gc", Platform:"linux/amd64"}
```

安装kind

```shell
curl -Lo ./kind https://github.com/kubernetes-sigs/kind/releases/download/v0.7.0/kind-$(uname)-amd64
chmod +x kind
mv kind /usr/local/bin
```

我们想要用docker provider的话，因为比较方便，作为deploy环境

```shell
cat > kind-cluster-with-extramounts.yaml <<EOF
kind: Cluster
apiVersion: kind.sigs.k8s.io/v1alpha3
nodes:
  - role: control-plane
    extraMounts:
      - hostPath: /var/run/docker.sock
        containerPath: /var/run/docker.sock
EOF
```

用Kind创建一个集群

```shell
[root@rain-k8s cluster_api]# kind create cluster --config ./kind-cluster-with-extramounts.yaml --name clusterapi
Creating cluster "clusterapi" ...
 ✓ Ensuring node image (kindest/node:v1.17.0) �
 ✓ Preparing nodes �
 ✓ Writing configuration �
 ✓ Starting control-plane �️
 ✓ Installing CNI �
 ✓ Installing StorageClass �
Set kubectl context to "kind-clusterapi"
You can now use your cluster with:

kubectl cluster-info --context kind-clusterapi

Not sure what to do next? � Check out https://kind.sigs.k8s.io/docs/user/quick-start/

[root@rain-k8s cluster_api]# kubectl cluster-info --context kind-clusterapi
Kubernetes master is running at https://127.0.0.1:32770
KubeDNS is running at https://127.0.0.1:32770/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

[root@rain-k8s cluster_api]# kubectl get node
NAME                       STATUS   ROLES    AGE    VERSION
clusterapi-control-plane   Ready    master   101s   v1.17.0
```

# 安装clusterctl

 ```shell
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v0.3.2/clusterctl-linux-amd64 -o clusterctl
chmod +x ./clusterctl
sudo mv ./clusterctl /usr/local/bin/clusterctl

[root@rain-k8s cluster_api]# clusterctl version
clusterctl version: &version.Info{Major:"0", Minor:"3", GitVersion:"v0.3.2-dirty", GitCommit:"867b2afed0e6a4509a74bb123b7c6f2ab0f7ffd7", GitTreeState:"dirty", BuildDate:"2020-03-19T19:29:46Z", GoVersion:"go1.13.8", Compiler:"gc", Platform:"linux/amd64"}

[root@rain-k8s cluster_api]# clusterctl config repositories
NAME          TYPE                     URL
cluster-api   CoreProvider             https://github.com/kubernetes-sigs/cluster-api/releases/latest/core-components.yaml
kubeadm       BootstrapProvider        https://github.com/kubernetes-sigs/cluster-api/releases/latest/bootstrap-components.yaml
kubeadm       ControlPlaneProvider     https://github.com/kubernetes-sigs/cluster-api/releases/latest/control-plane-components.yaml
aws           InfrastructureProvider   https://github.com/kubernetes-sigs/cluster-api-provider-aws/releases/latest/infrastructure-components.yaml
azure         InfrastructureProvider   https://github.com/kubernetes-sigs/cluster-api-provider-azure/releases/latest/infrastructure-components.yaml
metal3        InfrastructureProvider   https://github.com/metal3-io/cluster-api-provider-metal3/releases/latest/infrastructure-components.yaml
openstack     InfrastructureProvider   https://github.com/kubernetes-sigs/cluster-api-provider-openstack/releases/latest/infrastructure-components.yaml
vsphere       InfrastructureProvider   https://github.com/kubernetes-sigs/cluster-api-provider-vsphere/releases/latest/infrastructure-components.yaml

[root@rain-k8s cluster-api]# clusterctl config provider --core cluster-api
Name:               cluster-api
Type:               CoreProvider
URL:                https://github.com/kubernetes-sigs/cluster-api/releases/latest/core-components.yaml
Version:            v0.3.2
TargetNamespace:    capi-system
WatchingNamespace:
Images:
  - gcr.io/kubebuilder/kube-rbac-proxy:v0.4.1
  - us.gcr.io/k8s-artifacts-prod/cluster-api/cluster-api-controller:v0.3.2
  - gcr.io/kubebuilder/kube-rbac-proxy:v0.4.1
  - us.gcr.io/k8s-artifacts-prod/cluster-api/cluster-api-controller:v0.3.2
  
[root@rain-k8s cluster-api]# clusterctl config provider --bootstrap kubeadm
Name:               kubeadm
Type:               BootstrapProvider
URL:                https://github.com/kubernetes-sigs/cluster-api/releases/latest/bootstrap-components.yaml
Version:            v0.3.2
TargetNamespace:    capi-kubeadm-bootstrap-system
WatchingNamespace:
Images:
  - gcr.io/kubebuilder/kube-rbac-proxy:v0.4.1
  - us.gcr.io/k8s-artifacts-prod/cluster-api/kubeadm-bootstrap-controller:v0.3.2
  - gcr.io/kubebuilder/kube-rbac-proxy:v0.4.1
  - us.gcr.io/k8s-artifacts-prod/cluster-api/kubeadm-bootstrap-controller:v0.3.2
 ```

# docker provider

因为docker provider是测试用，所以要自己安装

先把cluster api代码clone下来

```shell
[root@rain-k8s cluster_api]# git clone https://github.com/kubernetes-sigs/cluster-api.git
[root@rain-k8s cluster_api]# cd cluster-api/
[root@rain-k8s cluster-api]# make -C test/infrastructure/docker docker-build REGISTRY=gcr.io/k8s-staging-capi-docker
[root@rain-k8s cluster-api]# make -C test/infrastructure/docker generate-manifests REGISTRY=gcr.io/k8s-staging-capi-docker
```

这一步会build一个docker provider的image

```shell
[root@rain-k8s cluster-api]# docker images | grep capd-manager-amd64
gcr.io/k8s-staging-capi-docker/capd-manager-amd64                dev                 132e0826fe6f        28 seconds ago       160MB
```

然后我们要把这个docker provider添加进cluster api

编辑~/.cluster-api/clusterctl.yaml

```shell
providers:
  - name: docker
    url: /root/.cluster-api/overrides/infrastructure-docker/latest/infrastructure-components.yaml
    type: InfrastructureProvider
```

```shell
[root@rain-k8s cluster-api]# clusterctl config repositories
NAME          TYPE                     URL
cluster-api   CoreProvider             https://github.com/kubernetes-sigs/cluster-api/releases/latest/core-components.yaml
kubeadm       BootstrapProvider        https://github.com/kubernetes-sigs/cluster-api/releases/latest/bootstrap-components.yaml
kubeadm       ControlPlaneProvider     https://github.com/kubernetes-sigs/cluster-api/releases/latest/control-plane-components.yaml
aws           InfrastructureProvider   https://github.com/kubernetes-sigs/cluster-api-provider-aws/releases/latest/infrastructure-components.yaml
azure         InfrastructureProvider   https://github.com/kubernetes-sigs/cluster-api-provider-azure/releases/latest/infrastructure-components.yaml
docker        InfrastructureProvider   /root/.cluster-api/overrides/infrastructure-docker/latest/infrastructure-components.yaml
metal3        InfrastructureProvider   https://github.com/metal3-io/cluster-api-provider-metal3/releases/latest/infrastructure-components.yaml
openstack     InfrastructureProvider   https://github.com/kubernetes-sigs/cluster-api-provider-openstack/releases/latest/infrastructure-components.yaml
vsphere       InfrastructureProvider   https://github.com/kubernetes-sigs/cluster-api-provider-vsphere/releases/latest/infrastructure-components.yaml
```

kind load make the docker provider image available for the kubelet in the management cluster.

```shell
[root@rain-k8s cluster-api]# kind load docker-image gcr.io/k8s-staging-capi-docker/capd-manager-amd64:dev --name clusterapi
Image: "gcr.io/k8s-staging-capi-docker/capd-manager-amd64:dev" with ID "sha256:132e0826fe6f265c65e96a50aaba210917c262530b4d994427bb5bd147a1a8f1" not present on node "clusterapi-control-plane"
```

安装go

```shell
wget https://studygolang.com/dl/golang/go1.14.1.linux-amd64.tar.gz
sudo tar -C /usr/local/ -xzvf 
```

编辑/etc/profile，在末尾加上

```shell
export GOROOT=/usr/local/go
export GOPATH=/home/ocp/goProject 
export GOBIN=$GOPATH/bin
export PATH=$PATH:$GOROOT/bin
export PATH=$PATH:$GOPATH/bin
```

source /etc/profile，立即生效

安装kustomize

```shell
go get github.com/kubernetes-sigs/kustomize
```

创建clusterctl-settings.json文件放在cluster api代码的根目录

```yaml
[root@rain-k8s cluster-api]# cat clusterctl-settings.json
{
    "providers": ["infrastructure-docker"]
}
```

run the local-overrides hack

```shell
[root@rain-k8s cluster-api]# cmd/clusterctl/hack/local-overrides.py
clusterctl local overrides generated from local repositories for the infrastructure-docker providers.
in order to use them, please run:

clusterctl init --infrastructure docker:v0.3.0

please check the documentation for additional steps required for using the docker provider
```

The script reads from the local repositories of the providers you want to install, builds the providers’ assets, and places them in a local override folder located under `$HOME/.cluster-api/overrides/`.

```shell
[root@rain-k8s cluster-api]# ls -al ~/.cluster-api/overrides/infrastructure-docker/v0.3.0/infrastructure-components.yaml
-rw-------. 1 root root 173 Mar 24 15:40 /root/.cluster-api/overrides/infrastructure-docker/v0.3.0/infrastructure-components.yaml
```

# 创建management cluster

The `clusterctl init` command installs the Cluster API components and transforms the Kubernetes cluster into a management cluster.

```shell
clusterctl init --infrastructure docker:v0.3.0

[root@rain-k8s ~]# kubectl get pods -A
NAMESPACE                           NAME                                                             READY   STATUS    RESTARTS   AGE
capd-system                         capd-controller-manager-7f544b5ccd-r524r                         2/2     Running   0          173m
capi-kubeadm-bootstrap-system       capi-kubeadm-bootstrap-controller-manager-6f64b78cc9-wv2gx       2/2     Running   0          15h
capi-kubeadm-control-plane-system   capi-kubeadm-control-plane-controller-manager-66475cc47f-wgjmn   2/2     Running   0          15h
capi-system                         capi-controller-manager-9c7899b86-v4w7z                          2/2     Running   0          15h
capi-webhook-system                 capi-controller-manager-68b5666f57-6r7qp                         2/2     Running   0          15h
capi-webhook-system                 capi-kubeadm-bootstrap-controller-manager-84cd9bb8dd-2h8k5       2/2     Running   0          15h
capi-webhook-system                 capi-kubeadm-control-plane-controller-manager-bfb887c9b-spgh2    2/2     Running   0          15h
```

这样我们的management cluster就创建好了，然后就可以用这个management cluster来创建workload cluster了

# 创建workload cluster

先生成一下workload cluster的yaml文件

```shell
[root@rain-k8s ~]# clusterctl config cluster my-cluster --kubernetes-version v1.17.2 --control-plane-machine-count=1 --worker-machine-count=1 > my-cluster.yaml
```

然后在my-cluster.yaml就是一份生成好的文件，可以修改，然后apply一下

```shell
[root@rain-k8s ~]# kubectl apply -f my-cluster.yaml
dockercluster.infrastructure.cluster.x-k8s.io/my-cluster created
cluster.cluster.x-k8s.io/my-cluster created
dockermachine.infrastructure.cluster.x-k8s.io/controlplane-0 created
machine.cluster.x-k8s.io/controlplane-0 created
kubeadmconfig.bootstrap.cluster.x-k8s.io/controlplane-0-config created
dockermachine.infrastructure.cluster.x-k8s.io/worker-0 created
machine.cluster.x-k8s.io/worker-0 created
kubeadmconfig.bootstrap.cluster.x-k8s.io/worker-0-config created

[root@rain-k8s ~]# kubectl get clusters
NAME         PHASE
my-cluster   Provisioned

[root@rain-k8s ~]# kubectl get machines
NAME             PROVIDERID                             PHASE
controlplane-0   docker:////my-cluster-controlplane-0   Running
worker-0         docker:////my-cluster-worker-0         Running

[root@rain-k8s ~]# docker ps
CONTAINER ID        IMAGE                          COMMAND                  CREATED             STATUS              PORTS                                  NAMES
b2d4b38a9ef9        kindest/node:v1.17.2           "/usr/local/bin/entr…"   3 minutes ago       Up 3 minutes                                               my-cluster-worker-0
c389beb8df30        kindest/node:v1.17.2           "/usr/local/bin/entr…"   3 minutes ago       Up 3 minutes        45499/tcp, 127.0.0.1:45499->6443/tcp   my-cluster-controlplane-0
8e393b722db1        kindest/haproxy:2.1.1-alpine   "/docker-entrypoint.…"   3 minutes ago       Up 3 minutes        36447/tcp, 0.0.0.0:36447->6443/tcp     my-cluster-lb
dc708210adab        kindest/node:v1.17.0           "/usr/local/bin/entr…"   19 hours ago        Up 19 hours         127.0.0.1:32770->6443/tcp              clusterapi-control-plane
```

cluster就建起来了

可以这样获取workload cluster的kubeconfig

```shell
[root@rain-k8s ~]# kubectl --namespace=default get secret/my-cluster-kubeconfig -o jsonpath={.data.value} \
| base64 --decode \
> ./my-cluster.kubeconfig
```

因为我们用的docker provider，还需要修改一下

```shell
[root@rain-k8s ~]# sed -i -e "s/server:.*/server: https:\/\/$(docker port my-cluster-lb 6443/tcp | sed "s/0.0.0.0/127.0.0.1/")/g" ./my-cluster.kubeconfig
[root@rain-k8s ~]# sed -i -e "s/certificate-authority-data:.*/insecure-skip-tls-verify: true/g" ./my-cluster.kubeconfig
```

然后就可以用kubeconfig查看新建的cluster了

```shell
[root@rain-k8s ~]# kubectl --kubeconfig=my-cluster.kubeconfig get nodes
NAME                        STATUS     ROLES    AGE   VERSION
my-cluster-controlplane-0   NotReady   master   37m   v1.17.2
my-cluster-worker-0         NotReady   <none>   37m   v1.17.2
```

control plane在我们还没安装CNI之前是不会ready的

我们安装cilium作为CNI

```shell
[root@rain-k8s ~]# kubectl --kubeconfig=my-cluster.kubeconfig apply -f https://raw.githubusercontent.com/cilium/cilium/v1.7/install/kubernetes/quick-install.yaml
serviceaccount/cilium created
serviceaccount/cilium-operator created
configmap/cilium-config created
clusterrole.rbac.authorization.k8s.io/cilium created
clusterrole.rbac.authorization.k8s.io/cilium-operator created
clusterrolebinding.rbac.authorization.k8s.io/cilium created
clusterrolebinding.rbac.authorization.k8s.io/cilium-operator created
daemonset.apps/cilium created
deployment.apps/cilium-operator created
```

# References

[1] [https://cluster-api.sigs.k8s.io/user/quick-start.html](https://cluster-api.sigs.k8s.io/user/quick-start.html)

[2] [https://cluster-api.sigs.k8s.io/clusterctl/developers.html](https://cluster-api.sigs.k8s.io/clusterctl/developers.html)