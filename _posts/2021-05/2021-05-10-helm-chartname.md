---
layout: post
title:  "helm安装时找不到包"
date:   2021-05-10 23:00:00 +0800
categories: Kubernetes
tags: Kubernetes-Deploy
excerpt: helm安装时找不到包
mathjax: true
typora-root-url: ../
---

# helm安装时找不到包

前几天在通过helm部署的时候发现总是找不到包

```shell
[root@rain-working-server ocp_onesearch]# helm search repo onesearch
NAME            	CHART VERSION	APP VERSION	DESCRIPTION
oke/ocponesearch	0.0.1        	1.0        	A Helm chart for OCPOneSearch Bot
```

明明search是可以搜索到的，但是安装的时候却总是提示找不到

helm install ocponesearch -f onesearch-values-prod.yaml oke/ocponesearch --debug -n iaas-ocp

后来才发现，原来helm安装的时候这个包的名字，需要跟我们在定义chart的时候名字对应起来才可以

```shell
[root@rain-working-server ocp_onesearch]# cat charts/ocp-onesearch/Chart.yaml
apiVersion: v1
appVersion: "1.0"
description: A Helm chart for OCPOneSearch Bot
name: OCPOneSearch
version: 0.0.1
```

我这边定义的时候，有大小写，就不一样了

```shell
[root@rain-working-server ocp_onesearch]# helm show chart oke/OCPOneSearch
apiVersion: v1
appVersion: "1.0"
description: A Helm chart for OCPOneSearch Bot
name: OCPOneSearch
version: 0.0.1
```

