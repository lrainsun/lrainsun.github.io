---
layout: post
title:  "Kubernetes RBAC"
date:   2019-11-01 22:03:00 +0800
categories: Go
tags: Kubernetes-RBAC
excerpt: Kubernetes里基于角色的权限控制
mathjax: true
typora-root-url: ../
---

# 基本概念

Kubernetes里的RBAC，反复提过的三个基本概念：

1. Role：角色，它其实是一组规则，定义了一组对 Kubernetes API 对象的操作权限。具体还有分：
   1. Role，作用于单个namespace
   2. ClusterRole，作用于cluster级别的，可以跨namespace
2. Subject：被作用者，既可以是“人”，也可以是“机器”，也可以使你在 Kubernetes 里
   定义的“用户”。subject具体定义的时候可以是：
   1. User：这是授权系统里的一个逻辑概念，用户需要通过外部认证服务，比如keystone
   2. Service Account：这也是一类user，可以认为是kubernetes内置的user
   3. Group：group其实代表了一组user
3. RoleBinding：定义了“被作用者”和“角色”的绑定关系。跟Role一样，rolebinding也有分：
   1. RoleBinding，作用于单个namespace
   2. ClusterRoleBinding，对整个集群所有namespace进行授权

# Role

创建一个role

```
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: test-rbac
  name: test-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

这里有几个定义：

* namespace代表了这个role只在test-rbac这个namespace里起作用

* apiGroups，这个呢其实是kubernetes的API Groups定义，属于哪个groups [https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.16]( https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.16) 这里可以查
  * core，常用的比如pod，container，configmap啥的都是core的
  * apps，deployment，daemonset啥的
  * batch，job啥的
  * ……这个写不完了，用的时候再查吧
  * 这里apiGroups定义为""，指的是默认的core
* resources就是具体的resource了，就是可以kubectl get **的那种，比如pods，deployments
* verbs就是具体可以执行的操作，get, list, watch, create, update, patch, delete

# ServiceAccount

创建一个ServiceAccount

```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: test-rbac
  name: rain-sa
```

创建了service account之后，kubernetes会对应创建一个token，这个token会被放到一个secret中在etcd里存放起来

```
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"ServiceAccount","metadata":{"annotations":{},"name":"rain-sa","namespace":"test-rbac"}}
  creationTimestamp: "2019-11-01T08:25:21Z"
  name: rain-sa
  namespace: test-rbac
  resourceVersion: "1111494"
  selfLink: /api/v1/namespaces/test-rbac/serviceaccounts/rain-sa
  uid: 12f5e9ae-e81d-4b3c-a48b-52fcc3892968
secrets:
- name: rain-sa-token-pz7p6
```

```
apiVersion: v1
data:
  ca.crt: ……
  namespace: dGVzdC1yYmFj
  token: ……
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: rain-sa
    kubernetes.io/service-account.uid: 12f5e9ae-e81d-4b3c-a48b-52fcc3892968
  creationTimestamp: "2019-11-01T08:25:21Z"
  name: rain-sa-token-pz7p6
  namespace: test-rbac
  resourceVersion: "1111493"
  selfLink: /api/v1/namespaces/test-rbac/secrets/rain-sa-token-pz7p6
  uid: 98e5af69-f6e0-4036-b7ca-4f00fa393d9d
type: kubernetes.io/service-account-token
```

可以看到token的data中包含了三个key，ca.cert，namespace，token，当pod使用了该service account后，就会把这个secret作为volume挂载到容器的/var/run/secrets/kubernetes.io/serviceaccount目录下，在projected volume里也提到过这个内容

# RoleBinding

rolebinding也是一个api对象，可以创建

其实就是把subjects跟role绑定起来

```
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: test-rolebinding
  namespace: test-rbac
subjects:
- kind: ServiceAccount
  name: rain-sa
  namespace: test-rbac
roleRef:
  kind: Role
  name: test-role
  apiGroup: rbac.authorization.k8s.io
```

# API Access

现在我们已经把role绑定到了rain-sa这个service account上，我们可以创建一个pod，使用这个service account，那么在这个pod里就可以获得role里定义的权限。

这里我们采用另外的方式来做验证。访问Kubernetes API可以用kubectl（使用kubeconfg做认证），还可以直接带上token访问API，我们都来试一下。

## kubeconfig

为了使用kubectl，我们需要根据刚刚的信息构造一个kubeconfig，依据service account来构造kubeconfig，需要包含以下信息：

* server ，也就是api service的IP
* cacert
* token
* namespace

```
server=https://10.225.28.220:6443
[root@rain-kubernetes-1 rain]# name=rain-sa-token-pz7p6
[root@rain-kubernetes-1 rain]# ca=$(kubectl get secret/$name -n test-rbac -o jsonpath='{.data.ca\.crt}')
[root@rain-kubernetes-1 rain]# token=$(kubectl get secret/$name -n test-rbac -o jsonpath='{.data.token}' | base64 --decode)
[root@rain-kubernetes-1 rain]# namespace=$(kubectl get secret/$name -n test-rbac -o jsonpath='{.data.namespace}' | base64 --decode)
[root@rain-kubernetes-1 rain]# echo "
> apiVersion: v1
> kind: Config
> clusters:
> - name: default-cluster
>   cluster:
>     certificate-authority-data: ${ca}
>     server: ${server}
> contexts:
> - name: default-context
>   context:
>     cluster: default-cluster
>     namespace: default
>     user: default-user
> current-context: default-context
> users:
> - name: default-user
>   user:
>     token: ${token}
> " > rain-sa.kubeconfig
```

```
apiVersion: v1
kind: Config
clusters:
- name: default-cluster
  cluster:
    certificate-authority-data: ……
    server: https://10.225.28.220:6443
contexts:
- name: default-context
  context:
    cluster: default-cluster
    namespace: default
    user: default-user
current-context: default-context
users:
- name: default-user
  user:
    token: ……
```

这样生成的一份Kubeconfig，就可以用来访问kubernetes API啦，试一下：

刚刚我们定义的rule是，可以get, list, watch pod资源：

```
[root@rain-kubernetes-1 rain]# kubectl --kubeconfig rain-sa.kubeconfig get pods -n test-rbac
NAME                                READY   STATUS    RESTARTS   AGE
ngnix-deployment-65c7fc84dc-5fl4f   1/1     Running   0          3s
ngnix-deployment-65c7fc84dc-8blhj   1/1     Running   0          3s
```

其他操作应该都是不被允许的：

```
[root@rain-kubernetes-1 rain]# kubectl --kubeconfig rain-sa.kubeconfig get secret -n test-rbac
Error from server (Forbidden): secrets is forbidden: User "system:serviceaccount:test-rbac:rain-sa" cannot list resource "secrets" in API group "" in the namespace "test-rbac"
```

```
[root@rain-kubernetes-1 rain]# kubectl --kubeconfig rain-sa.kubeconfig get secret --all-namespaces
Error from server (Forbidden): secrets is forbidden: User "system:serviceaccount:test-rbac:rain-sa" cannot list resource "secrets" in API group "" at the cluster scope

[root@rain-kubernetes-1 rain]# kubectl --kubeconfig rain-sa.kubeconfig get pods --all-namespaces
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:test-rbac:rain-sa" cannot list resource "pods" in API group "" at the cluster scope
```

cluster级别的访问也是不允许的，说明rule定义生效了

# token

还可以直接用curl的方式访问API

```
[root@rain-kubernetes-1 rain]# token=$(kubectl get secret/$name -n test-rbac -o jsonpath='{.data.token}' | base64 --decode)
[root@rain-kubernetes-1 rain]# ca=$(kubectl get secret/$name -n test-rbac -o jsonpath='{.data.ca\.crt}')
[root@rain-kubernetes-1 rain]# echo $ca | base64 -d >> ca.cert
```

```
[root@rain-kubernetes-1 rain]# curl --cacert ca.cert https://10.225.28.220:6443/api/v1/namespaces/test-rbac/pods -H "Authorization: Bearer ${token}"
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/namespaces/test-rbac/pods",
    "resourceVersion": "1128897"
  },
  "items": [
```

list pods成功咯

```
[root@rain-kubernetes-1 rain]# curl --cacert ca.cert https://10.225.28.220:6443/api/v1/namespaces/test-rbac/secrets -H "Authorization: Bearer ${token}"
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "secrets is forbidden: User \"system:serviceaccount:test-rbac:rain-sa\" cannot list resource \"secrets\" in API group \"\" in the namespace \"test-rbac\"",
  "reason": "Forbidden",
  "details": {
    "kind": "secrets"
  },
  "code": 403
```

list secrets失败了，因为我们没有赋这个权限

```
[root@rain-kubernetes-1 rain]# curl --cacert ca.cert https://10.225.28.220:6443/api/v1/pods -H "Authorization: Bearer ${token}"
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "pods is forbidden: User \"system:serviceaccount:test-rbac:rain-sa\" cannot list resource \"pods\" in API group \"\" at the cluster scope",
  "reason": "Forbidden",
  "details": {
    "kind": "pods"
  },
  "code": 403
}
```

list cluster level的pod也失败了

到目前为止，都是namespaced rbac，下面看下cluster level的

# ClusterRole

 ```
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: test-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
 ```

其实比Role就少了namespace的定义

# ClusterRoleBinding

```
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: test-clusterrolebinding
subjects:
- kind: ServiceAccount
  name: rain-sa
  namespace: test-rbac
roleRef:
  kind: ClusterRole
  name: test-clusterrole
  apiGroup: rbac.authorization.k8s.io
```

还是绑定到刚刚的service account上，token还是那个token，但是权限不一样了哦

还是测试一下：

```
[root@rain-kubernetes-1 rain]# kubectl --kubeconfig rain-sa.kubeconfig get pods --all-namespaces
NAMESPACE     NAME                                        READY   STATUS    RESTARTS   AGE
default       test-configmap-projected-volume             1/1     Running   1          33h
default       test-downwardapi-volume                     1/1     Running   0          27h
default       test-secrect-projected-volume               1/1     Running   1          33h
kube-system   coredns-5644d7b6d9-87f6c                    1/1     Running   0          9d
kube-system   coredns-5644d7b6d9-hvnvd                    1/1     Running   0          9d
kube-system   etcd-rain-kubernetes-1                      1/1     Running   0          9d
```

```
[root@rain-kubernetes-1 rain]# kubectl --kubeconfig rain-sa.kubeconfig get secrets --all-namespaces
Error from server (Forbidden): secrets is forbidden: User "system:serviceaccount:test-rbac:rain-sa" cannot list resource "secrets" in API group "" at the cluster scope
```

list cluster level pod成功

list cluster level secret失败

```
[root@rain-kubernetes-1 rain]#curl --cacert ca.cert https://10.225.28.220:6443/api/v1/pods -H "Authorization: Bearer ${token}" | less
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/pods",
    "resourceVersion": "1130567"
  },
  "items": [
```

```
[root@rain-kubernetes-1 rain]# curl --cacert ca.cert https://10.225.28.220:6443/api/v1/secrets -H "Authorization: Bearer ${token}"
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "secrets is forbidden: User \"system:serviceaccount:test-rbac:rain-sa\" cannot list resource \"secrets\" in API group \"\" at the cluster scope",
  "reason": "Forbidden",
  "details": {
    "kind": "secrets"
  },
  "code": 403
}
```

都符合预期，这就是RBAC啦

# User & Group

另外，除了service account和user，我觉得group的概念也很有必要学习一下

其实，一个service account，在kubernetes里对应的"用户"名字：

```
system:serviceaccount:<ServiceAccount 名字 >
```

而它对应的内置”用户组"的名字：

```
system:serviceaccounts:<Namespace 名字 >
```

通常，sytem: 前缀是系统保留的

```
subjects:
- kind: User
  name: Rain
  apiGroup: rbac.authorization.k8s.io
```

指定一个用户名为Rain的用户

```
subjects:
- kind: Group
  name: ocp
  apiGroup: rbac.authorization.k8s.io
```

指定一个名为ocp的组

```
subjects:
- kind: ServiceAccount
  name: default
  namespace: kube-system
```

指定kube-system namespace中默认的Service Account

```
subjects:
- kind: Group
  name: system:serviceaccounts:test-rbac
  apiGroup: rbac.authorization.k8s.io 
```

指定test-rbac namespace中全部的Service Account

```
subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io
```

指定全部namespace的全部Service Account

```
subjects:
- kind: Group
  name: system:authenticated
  apiGroup: rbac.authorization.k8s.io
```

指定全部的authenticated的用户

```
subjects:
- kind: Group
  name: system:unauthenticated
  apiGroup: rbac.authorization.k8s.io
```

指定全部的unauthenticated的用户

```
subjects:
- kind: Group
  name: system:authenticated
  apiGroup: rbac.authorization.k8s.io
- kind: Group
  name: system:unauthenticated
  apiGroup: rbac.authorization.k8s.io
```

指定全部的用户

另外，Kubernetes里内置了很多系统保留的ClusterRole，名字都是以system:开头，一般都是绑定给kubernetes系统组件对应的ServiceAccount使用的                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             

还有四个预定义好的ClusterRole

1. cluster-admin (整个kubernetes项目中最高权限)
2. admin
3. edit
4. view

# default ServiceAccount只读权限

如果我们不创建，默认的default service account会有什么权限呢？

我找了一下

```
[root@rain-kubernetes-1 rain]# kubectl get clusterrolebinding system:basic-user -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2019-10-23T11:42:39Z"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:basic-user
  resourceVersion: "98"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/system%3Abasic-user
  uid: 283ca194-20e1-47b9-83d1-21bba51ab459
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:basic-user
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:authenticated
```

看起来所有认证通过的都会被绑定到这个system:basic-user的role

```
[root@rain-kubernetes-1 rain]# kubectl get clusterrole system:basic-user -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2019-10-23T11:42:38Z"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:basic-user
  resourceVersion: "45"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterroles/system%3Abasic-user
  uid: fc953b91-850f-4220-af44-4e64649da97a
rules:
- apiGroups:
  - authorization.k8s.io
  resources:
  - selfsubjectaccessreviews
  - selfsubjectrulesreviews
  verbs:
  - create
```

selfsubjectaccessreviews & selfsubjectrulesreviews还整不明白。。。

```
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: default-readonly-binding
subjects:
  - kind: Group
    name: system:serviceaccounts
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

但是这样是所有namespace下所有serviceaccount都只读权限了，而且可以访问所有namespace下的resource

怎么只让default ServiceAccount只读，且只能访问本namespace的呢？待解决。。。