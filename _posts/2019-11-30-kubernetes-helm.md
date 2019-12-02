---
layout: post
title:  "Kubernetes Helm包管理"
date:   2019-11-30 15:55:00 +0800
categories: Kubernetes
tags: Kubernetes-Deploy
excerpt: Kubernetes Helm包管理
mathjax: true
typora-root-url: ../
---

[Helm](http://helm.sh/)是一个kubernetes应用的包管理工具，用来管理[charts](https://github.com/kubernetes/charts)——预先配置好的安装包资源，有点类似于Ubuntu的APT和CentOS中的yum。

Helm chart是用来封装kubernetes原生应用程序的yaml文件，可以在你部署应用的时候自定义应用程序的一些metadata，便与应用程序的分发。

Helm和charts的主要作用：

- 应用程序封装
- 版本管理
- 依赖检查
- 便于应用程序分发

Helm 有三个重要概念：

- chart：包含了创建`Kubernetes`的一个应用实例的必要信息
- config：包含了应用发布配置信息
- release：是一个 chart 及其配置的一个运行实例

# 安装helm客户端

```shell
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
```

创建tiller的`serviceaccount`和`clusterrolebinding`

```shell
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
```

安装helm服务器端tiller

```shell
[root@rain-kubernetes-1 helm]# helm init

[root@rain-kubernetes-1 helm]# kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'

[root@rain-kubernetes-1 helm]# kubectl -n kube-system get pods | grep tiller
tiller-deploy-969865475-jgrlj               1/1     Running            0          55m

[root@rain-kubernetes-1 helm]# helm version
Client: &version.Version{SemVer:"v2.16.1", GitCommit:"bbdfe5e7803a12bbdf97e94cd847859890cf4050", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.16.1", GitCommit:"bbdfe5e7803a12bbdf97e94cd847859890cf4050", GitTreeState:"clean"}
```

# 创建chart

```shell
[root@rain-kubernetes-1 helm]# tree mangodb/
mangodb/
├── charts   #依赖的chart
├── Chart.yaml   #Chart本身的版本和配置信息
├── templates     #配置模板目录
│   ├── deployment.yaml   #kubernetes Deployment object
│   ├── _helpers.tpl   #用于修改kubernetes objcet配置的模板
│   ├── ingress.yaml   #kubernetes Ingress
│   ├── NOTES.txt    #helm提示信息
│   ├── serviceaccount.yaml   #kubernetes ServiceAccount
│   ├── service.yaml   #kubernetes Serivce
│   └── tests
│       └── test-connection.yaml
└── values.yaml

3 directories, 9 files
```

## 模板

`Templates`目录下是yaml文件的模板，遵循[Go template](https://golang.org/pkg/text/template/)语法。

比如service模板：

```yaml
[root@rain-kubernetes-1 mangodb]# cat templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: \{\{ include "mangodb.fullname" . \}\}
  labels:
\{\{ include "mangodb.labels" . | indent 4 \}\}
spec:
  type: \{\{ .Values.service.type \}\}
  ports:
    - port: \{\{ .Values.service.port \}\\}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app.kubernetes.io/name: \{\{ include "mangodb.name" . \}\}
    app.kubernetes.io/instance: \{\{ .Release.Name \}\}
```

类似`mangodb.fullname`和`mangodb.labels`这样的变量是在`templates/_helpers.tpl`里定义的

```shell
[root@rain-kubernetes-1 mangodb]# grep "mangodb.fullname" -rn templates/_helpers.tpl
14:\{\{- define "mangodb.fullname" -\}\}
```

而`.Values.service.port`则是在values.yaml里定义的

```yaml
service:
  type: ClusterIP
  port: 80
```

其他模板定义也都类似

## 检查配置和模板是否有效

当使用kubernetes部署应用的时候实际上讲templates渲染成最终的kubernetes能够识别的yaml格式。

使用`helm install --dry-run --debug `命令来验证chart配置。

```shell
[root@rain-kubernetes-1 helm]# helm install --dry-run --debug mangodb
```

该输出中包含了模板的变量配置与最终渲染的yaml文件。

# 部署chart

```shell
[root@rain-kubernetes-1 mangodb]# helm install .
NAME:   lucky-lobster
LAST DEPLOYED: Sat Nov 30 04:30:09 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Deployment
NAME                   AGE
lucky-lobster-mangodb  1s

==> v1/Pod(related)
NAME                                    AGE
lucky-lobster-mangodb-544cdc5f99-tn257  1s

==> v1/Service
NAME                   AGE
lucky-lobster-mangodb  1s

==> v1/ServiceAccount
NAME                   AGE
lucky-lobster-mangodb  1s


NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=mangodb,app.kubernetes.io/instance=lucky-lobster" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl port-forward $POD_NAME 8080:80
  
[root@rain-kubernetes-1 mangodb]# kubectl get service | grep mangodb
lucky-lobster-mangodb   ClusterIP      10.101.224.108   <none>                  80/TCP           3m6s
[root@rain-kubernetes-1 mangodb]# curl 10.101.224.108:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

查看部署的release

```shell
[root@rain-kubernetes-1 mangodb]# helm list
NAME            REVISION        UPDATED                         STATUS          CHART           APP VERSION     NAMESPACE
lucky-lobster   1               Sat Nov 30 04:30:09 2019        DEPLOYED        mangodb-0.1.0   1.0             default
```

删除部署的release

```shell
[root@rain-kubernetes-1 mangodb]# helm delete lucky-lobster
release "lucky-lobster" deleted
```

# 打包分享

除此之外，还可以打包chart，通过修改Chart.yaml可以修改chart配置

```yaml
[root@rain-kubernetes-1 mangodb]# cat Chart.yaml
apiVersion: v1
appVersion: "1.0"
description: A Helm chart for Kubernetes
name: mangodb
version: 0.1.0
```

还可以在requirements里定义应用依赖的chart

```yaml
[root@rain-kubernetes-1 mangodb]# cat requirments.yaml
dependencies:
- name: mariadb
  version: 0.6.0
  repository: https://kubernetes-charts.storage.googleapis.com
```

使用helm lint可以检查依赖和模板配置是否正确

```shell
[root@rain-kubernetes-1 mangodb]# helm lint .
==> Linting .
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, no failures
```

## 安装源

我们可以把做好chart包上传到helm repo，首先需要添加helm库

```shell
[root@rain-kubernetes-1 mangodb]# helm repo add oke https://hlm.prv.webex.com/
```

helm search可以搜索repo中提供的helm chart

```shell
[root@rain-kubernetes-1 mangodb]# helm search
```

## 打包

helm package进行打包

```shell
[root@rain-kubernetes-1 mangodb]# helm package ./
Successfully packaged chart and saved it to: /root/rain/helm/mangodb/mangodb-0.1.0.tgz
```

## 发布

如果使用`helm push`需要先安装插件

```shell
[root@rain-kubernetes-1 mangodb]# helm plugin install https://github.com/chartmuseum/helm-push
Downloading and installing helm-push v0.7.1 ...
https://github.com/chartmuseum/helm-push/releases/download/v0.7.1/helm-push_0.7.1_linux_amd64.tar.gz
Installed plugin: push

[root@rain-kubernetes-1 helm]# helm push mangodb/ oke
Pushing mangodb-0.1.0.tgz to oke...
Done.
```

需要update一下，就能看到我们push的chart了

```shell
[root@rain-kubernetes-1 helm]# helm update
Command "update" is deprecated, Use 'helm repo update'

Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "oke" chart repository
...Successfully got an update from the "fabric8" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete.
[root@rain-kubernetes-1 helm]# helm search mangodb
NAME            CHART VERSION   APP VERSION     DESCRIPTION
local/mangodb   0.1.0           1.0             A Helm chart for Kubernetes
oke/mangodb     0.1.0           1.0             A Helm chart for Kubernetes
```

除了`helm push`还可以使用`curl`来上传chart到repo，重新打包一个0.1.1，再上传

```shell
[root@rain-kubernetes-1 mangodb]# helm package ./
Successfully packaged chart and saved it to: /root/rain/helm/mangodb/mangodb-0.1.1.tgz
[root@rain-kubernetes-1 mangodb]# curl --data-binary "@mangodb-0.1.1.tgz" https://hlm.prv.webex.com/api/charts
{"saved":true}[root@rain-kubernetes-1 mangodb]# helm update
Command "update" is deprecated, Use 'helm repo update'

Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "oke" chart repository
...Successfully got an update from the "fabric8" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete.
[root@rain-kubernetes-1 mangodb]# helm search mangodb
NAME            CHART VERSION   APP VERSION     DESCRIPTION
local/mangodb   0.1.1           1.0             A Helm chart for Kubernetes
oke/mangodb     0.1.1           1.0             A Helm chart for Kubernetes
```

或者通过这样的方式，直接指定version

```shell
[root@rain-kubernetes-1 helm]# helm push mangodb oke --version=0.1.2
Pushing mangodb-0.1.2.tgz to oke...
Done.
```

## 部署

现在就可以从helm repo部署我们的应用了

```shell
[root@rain-kubernetes-1 mangodb]# helm install oke/mangodb
NAME:   killjoy-penguin
LAST DEPLOYED: Sat Nov 30 08:36:34 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Deployment
NAME                     AGE
killjoy-penguin-mangodb  0s

==> v1/Pod(related)
NAME                                      AGE
killjoy-penguin-mangodb-7bbbc57bbd-gk5wv  0s

==> v1/Service
NAME                     AGE
killjoy-penguin-mangodb  0s

==> v1/ServiceAccount
NAME                     AGE
killjoy-penguin-mangodb  0s


NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=mangodb,app.kubernetes.io/instance=killjoy-penguin" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl port-forward $POD_NAME 8080:80
  
[root@rain-kubernetes-1 mangodb]# helm list --all
NAME            REVISION        UPDATED                         STATUS          CHART           APP VERSION     NAMESPACE
killjoy-penguin 1               Sat Nov 30 08:36:34 2019        DEPLOYED        mangodb-0.1.2   1.0             default
lucky-lobster   1               Sat Nov 30 04:30:09 2019        DELETED         mangodb-0.1.0   1.0             default

[root@rain-kubernetes-1 mangodb]# helm status killjoy-penguin
LAST DEPLOYED: Sat Nov 30 08:36:34 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Deployment
NAME                     AGE
killjoy-penguin-mangodb  7m35s

==> v1/Pod(related)
NAME                                      AGE
killjoy-penguin-mangodb-7bbbc57bbd-gk5wv  7m35s

==> v1/Service
NAME                     AGE
killjoy-penguin-mangodb  7m35s

==> v1/ServiceAccount
NAME                     AGE
killjoy-penguin-mangodb  7m35s


NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=mangodb,app.kubernetes.io/instance=killjoy-penguin" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl port-forward $POD_NAME 8080:80
```

也可以指定安装的version

```shell
[root@rain-kubernetes-1 mangodb]# helm install oke/mangodb --version 0.1.1
```

## 更新release

```shell
[root@rain-kubernetes-1 mangodb]# helm list
NAME            REVISION        UPDATED                         STATUS          CHART           APP VERSION     NAMESPACE
killjoy-penguin 1               Sat Nov 30 08:36:34 2019        DEPLOYED        mangodb-0.1.2   1.0             default

[root@rain-kubernetes-1 mangodb]# helm upgrade killjoy-penguin -f values.yaml --set service.port=81 oke/mangodb
Release "killjoy-penguin" has been upgraded.

[root@rain-kubernetes-1 mangodb]# kubectl get service | grep mango
killjoy-penguin-mangodb   ClusterIP      10.102.130.51   <none>                  81/TCP           15m
```

可以看到service port已经被修改成81了

## 删除库上的chart

可以通过curl来删除

```shell
[root@rain-kubernetes-1 mangodb]# curl -X DELETE https://hlm.prv.webex.com/api/charts/mangodb/0.1.2
{"deleted":true}

[root@rain-kubernetes-1 mangodb]# helm update
Command "update" is deprecated, Use 'helm repo update'

Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "oke" chart repository
...Successfully got an update from the "stable" chart repository
...Successfully got an update from the "fabric8" chart repository
Update Complete.

[root@rain-kubernetes-1 mangodb]# helm search mangodb
NAME            CHART VERSION   APP VERSION     DESCRIPTION
local/mangodb   0.1.1           1.0             A Helm chart for Kubernetes
oke/mangodb     0.1.1           1.0             A Helm chart for Kubernetes
```

可以根据version来搜索chart

```shell
[root@rain-kubernetes-1 mangodb]# helm search mangodb --version 0.1.1
NAME            CHART VERSION   APP VERSION     DESCRIPTION
local/mangodb   0.1.1           1.0             A Helm chart for Kubernetes
oke/mangodb     0.1.1           1.0             A Helm chart for Kubernetes

[root@rain-kubernetes-1 mangodb]# helm search mangodb --version 0.1.2
No results found
```

