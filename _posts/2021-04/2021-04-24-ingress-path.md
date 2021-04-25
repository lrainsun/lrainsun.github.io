---
layout: post
title:  "Kubernetes Ingress相对路径"
date:   2021-04-24 23:00:00 +0800
categories: Kubernetes
tags: Kubernetes-Service
excerpt: Kubernetes Ingress相对路径
mathjax: true
typora-root-url: ../
---

# Kubernetes Ingress相对路径

我们有一个dynamic parameter service的服务，想要通过ingress配置到 [http://ocpadmin.prv.webex.com/api/v1/dynamic_parameters](http://ocpadmin.prv.webex.com/api/v1/dynamic_parameters/OCP4xCreateDeployment)

首先创建一个dynamic parameter的service

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    meta.helm.sh/release-name: dynamic-parameter
    meta.helm.sh/release-namespace: iaas-ocp
  creationTimestamp: "2021-04-23T08:36:39Z"
  labels:
    app.kubernetes.io/instance: dynamic-parameter
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: dynamic-parameter
    helm.sh/chart: dynamic-parameter-0.0.8
  name: dynamic-parameter
  namespace: iaas-ocp
  resourceVersion: "376309823"
  selfLink: /api/v1/namespaces/iaas-ocp/services/dynamic-parameter
  uid: b433300b-4550-4bc2-aaee-b34e9fd88b7a
spec:
  clusterIP: 10.98.122.233
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 30424
    port: 8080
    protocol: TCP
    targetPort: 5000
  selector:
    app.kubernetes.io/instance: dynamic-parameter
    app.kubernetes.io/name: dynamic-parameter
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```

然后创建Ingress

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: iaas-ocp-ingress
spec:
  rules:
  - http:
      paths:
      - path: /api/v1/dynamic_parameters
        backend:
          serviceName: dynamic-parameter
          servicePort: 8080
```

这时候发现api访问是有问题的，因为到dynamic_parameters的请求路径是绝对路径，就是带上/api/v1/dynamic_parameters的

而在我们的代码里是没有前面的路径的

```python
api.add_resource(DynamicParameter, '/<pipeline>')
api.add_resource(DynamicParameterSchema, '/schema/<pipeline>')
api.add_resource(HealthCheck, '/healthz')
```

这个要在Ingress里配置相对路径

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: iaas-ocp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - http:
      paths:
      - path: /api/v1/dynamic_parameters(/|$)(.*)
        backend:
          serviceName: dynamic-parameter
          servicePort: 8080
```

在 `path` 中通过正则表达式 `/app(/|$)(.*)` ，其实是将匹配的路径设置成了 `rewrite-target` 的目标路径了

