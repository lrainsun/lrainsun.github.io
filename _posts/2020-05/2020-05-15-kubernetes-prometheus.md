---
layout: post
title:  "为kubernetes环境增加监控"
date:   2020-05-15 23:01:00 +0800
categories: Kubernetes
tags: Kubernetes-Monitor
excerpt: 为kubernetes环境增加监控
mathjax: true
typora-root-url: ../
---

# 部署prometheus-operator

之前学习过prometheus-operator，嗯，差不多忘了😅[prometheus-operator](https://lrainsun.github.io/2019/11/28/prometheus-operator/)

有更简单的部署方式是通过helm

```shell
helm repo add stable https://kubernetes-charts.storage.googleapis.com/

helm search repo prometheus-operator

NAME                          CHART VERSION    APP VERSION    DESCRIPTION

stable/prometheus-operator    8.13.7           0.38.1         Provides easy monitoring definitions for Kubern...

helm --kubeconfig ~/Documents/k8s/airflow install prometheus-operator stable/prometheus-operator -n monitoring
```

部署完之后，看到有这么多东西

```shell
kubectl --kubeconfig ~/Documents/k8s/airflow --namespace monitoring get all
NAME                                                         READY   STATUS    RESTARTS   AGE
pod/alertmanager-prometheus-operator-alertmanager-0          2/2     Running   0          59m
pod/prometheus-operator-grafana-64bcbf975f-62jkc             2/2     Running   0          59m
pod/prometheus-operator-kube-state-metrics-5fdcd78bc-sbp8w   1/1     Running   0          59m
pod/prometheus-operator-operator-5dd8f8f568-4x9hd            2/2     Running   0          59m
pod/prometheus-operator-prometheus-node-exporter-h28px       1/1     Running   0          59m
pod/prometheus-prometheus-operator-prometheus-0              3/3     Running   1          59m

NAME                                                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/alertmanager-operated                          ClusterIP   None            <none>        9093/TCP,9094/TCP,9094/UDP   59m
service/prometheus-operated                            ClusterIP   None            <none>        9090/TCP                     59m
service/prometheus-operator-alertmanager               ClusterIP   10.101.98.91    <none>        9093/TCP                     59m
service/prometheus-operator-grafana                    ClusterIP   10.103.94.136   <none>        80/TCP                       59m
service/prometheus-operator-kube-state-metrics         ClusterIP   10.96.53.79     <none>        8080/TCP                     59m
service/prometheus-operator-operator                   ClusterIP   10.108.161.70   <none>        8080/TCP,443/TCP             59m
service/prometheus-operator-prometheus                 ClusterIP   10.108.115.91   <none>        9090/TCP                     59m
service/prometheus-operator-prometheus-node-exporter   ClusterIP   10.98.177.65    <none>        9100/TCP                     59m

NAME                                                          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/prometheus-operator-prometheus-node-exporter   1         1         1       1            1           <none>          59m

NAME                                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/prometheus-operator-grafana              1/1     1            1           59m
deployment.apps/prometheus-operator-kube-state-metrics   1/1     1            1           59m
deployment.apps/prometheus-operator-operator             1/1     1            1           59m

NAME                                                               DESIRED   CURRENT   READY   AGE
replicaset.apps/prometheus-operator-grafana-64bcbf975f             1         1         1       59m
replicaset.apps/prometheus-operator-kube-state-metrics-5fdcd78bc   1         1         1       59m
replicaset.apps/prometheus-operator-operator-5dd8f8f568            1         1         1       59m

NAME                                                             READY   AGE
statefulset.apps/alertmanager-prometheus-operator-alertmanager   1/1     59m
statefulset.apps/prometheus-prometheus-operator-prometheus       1/1     59m

MINSU-M-M1RW:k8s minsu$ kubectl --kubeconfig ~/Documents/k8s/airflow get prometheus -n monitoring
NAME                             VERSION   REPLICAS   AGE
prometheus-operator-prometheus   v2.17.2   1          6h20m
MINSU-M-M1RW:k8s minsu$ kubectl --kubeconfig ~/Documents/k8s/airflow get alertmanager -n monitoring
NAME                               VERSION   REPLICAS   AGE
prometheus-operator-alertmanager   v0.20.0   1          6h20m
MINSU-M-M1RW:k8s minsu$ kubectl --kubeconfig ~/Documents/k8s/airflow get servicemonitor -n monitoring
NAME                                          AGE
prometheus-operator-alertmanager              6h20m
prometheus-operator-apiserver                 6h20m
prometheus-operator-coredns                   6h20m
prometheus-operator-grafana                   6h20m
prometheus-operator-kube-controller-manager   6h20m
prometheus-operator-kube-etcd                 6h20m
prometheus-operator-kube-proxy                6h20m
prometheus-operator-kube-scheduler            6h20m
prometheus-operator-kube-state-metrics        6h20m
prometheus-operator-kubelet                   6h20m
prometheus-operator-node-exporter             6h20m
prometheus-operator-operator                  6h20m
prometheus-operator-prometheus                6h20m
MINSU-M-M1RW:k8s minsu$ kubectl --kubeconfig ~/Documents/k8s/airflow get prometheusrule -n monitoring
NAME                                                       AGE
prometheus-operator-alertmanager.rules                     6h21m
prometheus-operator-etcd                                   6h21m
prometheus-operator-general.rules                          6h21m
prometheus-operator-k8s.rules                              6h21m
prometheus-operator-kube-apiserver-slos                    6h21m
prometheus-operator-kube-apiserver.rules                   6h21m
prometheus-operator-kube-prometheus-general.rules          6h21m
prometheus-operator-kube-prometheus-node-recording.rules   6h21m
prometheus-operator-kube-scheduler.rules                   6h21m
prometheus-operator-kube-state-metrics                     6h21m
prometheus-operator-kubelet.rules                          6h21m
prometheus-operator-kubernetes-apps                        6h21m
prometheus-operator-kubernetes-resources                   6h21m
prometheus-operator-kubernetes-storage                     6h21m
prometheus-operator-kubernetes-system                      6h21m
prometheus-operator-kubernetes-system-apiserver            6h21m
prometheus-operator-kubernetes-system-controller-manager   6h21m
prometheus-operator-kubernetes-system-kubelet              6h21m
prometheus-operator-kubernetes-system-scheduler            6h21m
prometheus-operator-node-exporter                          6h21m
prometheus-operator-node-exporter.rules                    6h21m
prometheus-operator-node-network                           6h21m
prometheus-operator-node.rules                             6h21m
prometheus-operator-prometheus                             6h21m
prometheus-operator-prometheus-operator                    6h21m
```

# 部署ingress controller

可以看到service都用的clusterip，对外暴露服务的话，要不改成Nodeport形式，或者我们还需要用到ingress controller

```shell
kubectl --kubeconfig ~/Documents/k8s/airflow apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-0.32.0/deploy/static/provider/cloud/deploy.yaml
```

检查一下ingress-nginx-controller是running的，ingress-nginx-controller service把ingress-nginx-controller通过loadbalancer的方式暴露到外部，我没安装外部Lb，所以external-ip还没有

```shell
MINSU-M-M1RW:bin minsu$ kubectl --kubeconfig ~/Documents/k8s/airflow get pods -n ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-p8zrg        0/1     Completed   0          4m53s
ingress-nginx-admission-patch-59926         0/1     Completed   0          4m53s
ingress-nginx-controller-866488c6d4-wtm4q   1/1     Running     0          5m3s

MINSU-M-M1RW:bin minsu$ kubectl --kubeconfig ~/Documents/k8s/airflow get service -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.97.138.112   <pending>     80:30220/TCP,443:32145/TCP   27m
ingress-nginx-controller-admission   ClusterIP      10.101.164.10   <none>        443/TCP                      27m
```

# 创建ingress

grafana, prometheus, alertmanager的service都已经有了，只需要创建ingress就可以了

```shell
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: grarana-ingress
  namespace: monitoring
spec:
  backend:
    serviceName: prometheus-operator-grafana
    servicePort: 80
```

apply之后，就可以Ingress的后端就是grafana了

```shell
MINSU-M-M1RW:k8s minsu$ kubectl --kubeconfig ~/Documents/k8s/airflow apply -f prometheus-ingress.yaml
ingress.extensions/grarana-ingress created
MINSU-M-M1RW:k8s minsu$ kubectl --kubeconfig ~/Documents/k8s/airflow get ingress -n monitoring
NAME              CLASS    HOSTS   ADDRESS   PORTS   AGE
grarana-ingress   <none>   *                 80      24s
MINSU-M-M1RW:k8s minsu$ kubectl --kubeconfig ~/Documents/k8s/airflow describe ingress -n monitoring
Name:             grarana-ingress
Namespace:        monitoring
Address:
Default backend:  prometheus-operator-grafana:80 (10.244.0.71:3000)
Rules:
  Host  Path  Backends
  ----  ----  --------
  *     *     prometheus-operator-grafana:80 (10.244.0.71:3000)
Annotations:
  kubectl.kubernetes.io/last-applied-configuration:  {"apiVersion":"extensions/v1beta1","kind":"Ingress","metadata":{"annotations":{},"name":"grarana-ingress","namespace":"monitoring"},"spec":{"backend":{"serviceName":"prometheus-operator-grafana","servicePort":80}}}

Events:
  Type    Reason  Age   From                      Message
  ----    ------  ----  ----                      -------
  Normal  CREATE  93s   nginx-ingress-controller  Ingress monitoring/grarana-ingress
```

然后可以通过nginx-ingress-controller来访问，现在nginx-ingress-controller的service是lb模式，没配外部lb，所以暂时用nodeport来访问

![image-20200515153720798](/../assets/images/image-20200515153720798.png)

那还想把alertmanager和prometheus也暴露出来，可以用同一url不同路径，改一下

```shell
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: prometheus-ingress
  namespace: monitoring
spec:

  rules:
  - http:
      paths:
      - path: /grafana
        backend:
          serviceName: prometheus-operator-grafana
          servicePort: 80
      - path: /alertmanager
        backend:
          serviceName: prometheus-operator-alertmanager
          servicePort: 9093
      - path: /prometheus
        backend:
          serviceName: prometheus-operator-prometheus
          servicePort: 9090
```

结果404了

![image-20200515160316074](/../assets/images/image-20200515160316074.png)

题外话：通过`kubectl --kubeconfig ~/Documents/k8s/airflow exec ingress-nginx-controller-866488c6d4-wtm4q -n ingress-nginx -- cat /etc/nginx/nginx.conf`可以查看nginx的配置

需要edit promethues和alertmanager crd，把

```shell
externalUrl: http:///alertmanager
=>
routePrefix: /alertmanager
```

grafana改起来比较麻烦，所以直接映射到/

```shell
  rules:
  - http:
      paths:
      - backend:
          serviceName: prometheus-operator-grafana
          servicePort: 80
        path: /
        pathType: ImplementationSpecific
      - backend:
          serviceName: prometheus-operator-alertmanager
          servicePort: 9093
        path: /alertmanager
        pathType: ImplementationSpecific
      - backend:
          serviceName: prometheus-operator-prometheus
          servicePort: 9090
        path: /prometheus
        pathType: ImplementationSpecific
```

这样改了之后就可以了

![image-20200515173325703](/../assets/images/image-20200515173325703.png)

![image-20200515173437537](/../assets/images/image-20200515173437537.png)

![image-20200515173458595](/../assets/images/image-20200515173458595.png)

