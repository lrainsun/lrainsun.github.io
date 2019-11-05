---
layout: post
title:  "Kubernetes CRD自定义资源"
date:   2019-11-05 22:12:00 +0800
categories: Kubernetes
tags: Kubernetes-Controller
excerpt: Kubernetes CRD自定义资源
mathjax: true
typora-root-url: ../
---

# CRD定义

CRD 的全称是 Custom Resource Definition。顾名思义，它指的就是，允许用户在Kubernetes 中添加一个跟 Pod、Node 类似的、新的 API 资源类型，即：自定义 API 资源。

前面控制器实现了API对象的业务逻辑，而CRD可以自定义API资源。

我有一点点懒，先把概念记在这里，等到要用的时候再自己动手实践。嗯。。貌似是篇比较水的blog，O(∩_∩)O哈哈~ 强迫症表示这个需要记，所以没办法跳过，但是又不想现在实践

CRD的定义包括哪些内容：

1. 前面讲过，一个API对象在Etcd里的完整资源路径，是由：Group（API组）、Version（API版本）和Resource（API资源类型）三个部分组成的。那么在定义CRD的时候也要定义着三个内容。——> CustomResourceDefinition定义
2. 自定义的对象长啥样，也就是对象描述，spec, status等 ——> 通过增加kubernetes代码

## CustomResourceDefinition

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: networks.samplecrd.k8s.io # spec.names.plural+"."+spec.group
spec:
  group: samplecrd.k8s.io # crd api group
  version: v1 # crd api version 
  names:
    kind: Network  # crd api resource
    plural: networks  # crd api resource plural
  scope: Namespaced
```

## Code Change

```shell
$ tree $GOPATH/src/github.com/<your-name>/k8s-controller-custom-resource
.
├── controller.go
├── crd
│ └── network.yaml
├── example
│ └── example-network.yaml
├── main.go
└── pkg
	└── apis
		└── samplecrd  # pkg/apis/samplecrd crd api group name
			├── register.go
			└── v1   # crd api version
				├── doc.go # 定义在 doc.go文件的注释，起到的是全局的代码生成控制的作用，也被称为Global Tags
				├── register.go  # 放置全局变量，如groupname, verison等
				└── types.go  # 定义了network对象的完整描述，spec字段里的内容
```

然后kubernetes有提供代码生成工具，自动生成clientset，informer和lister，这些都会被控制器用到哒

## 创建CRD对象

上面都搞定之后，就可以用类似这样的yaml来创建对象啦 

```
apiVersion: samplecrd.k8s.io/v1
kind: Network
metadata:
  name: example-network
spec:
  cidr: "192.168.0.0/16"
  gateway: "192.168.0.1"
```

