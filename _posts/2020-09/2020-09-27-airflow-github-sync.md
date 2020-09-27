---
layout: post
title:  "同步github下的dag到airflow"
date:   2020-09-27 23:00:00 +0800
categories: Airflow
tags: Airflow-Tutorial
excerpt: 同步github下的dag到airflow
mathjax: true
typora-root-url: ../
---

# 同步github下的dag到airflow

airflow的官方[Helm chart](https://github.com/helm/charts/blob/master/stable/airflow)有啦，通过这个可以快速在kubernetes上部署airflow，装了最新的1.10.12版本，发现，在1.10.10版本下，scheduler跟webserver是放在一个pod里的，在1.10.12被分成两个pod了。

而image的build，我们之前是直接拿airflow代码下来，增加东西直接build的

其实完全可以

```dockerfile
FROM apache/airflow:1.10.12-python3.6
```

然后再增加我们自己的内容，比如添加内容，安装新的包啥的

测试的时候我们是把/opt/airflow/dags mount出来，然后手工拷贝dag过去的，其实是可以从github上同步dag下来的

配置大概是这样，helm chart的values里这样配置

```yaml
dags:
  path: /opt/airflow/dags
  git:
    url: 'git@wwwin-github.cisco.com:webex-iaas/ocp_dags.git'
    ref: dags
    secret: airflow-secrets
    privateKeyName: gitSshKey
    repoHost: 'wwwin-github.cisco.com'
    gitSync:
      enabled: true
      refreshTime: 60
      images:
        repository: alpine/git
        tag: latest
        pullPolicy: IfNotPresent
    initContainer:
      enabled: true
    ## Image for the init container (any image with git will do)
      image:
        ## docker-airflow image
        repository: k8s.gcr.io/git-sync
        ## image tag
        tag: v3.1.1
        ## Image pull policy
        ## values: Always or IfNotPresent
        pullPolicy: IfNotPresent
      ## install requirements.txt dependencies automatically
      installRequirements: true
```

initContainer就是在webserver & scheduler容器没起来之前，先去git clone一下dags的branch，而gitsync会每隔一段时间就去sync github上的内容

使用ssh登录的前提是，要先创建一个secret，在这里名为airflow-secrets，里面包含的内容是key为gitSshKey，内容是可以用来访问github的private key，而为了host key checking，还需要包含名为known_hosts，内容可以这样查`ssh-keyscan wwwin-github.cisco.com`

而除了使用ssh的方式，还可以采用https的方式

```yaml
dags:
  path: /opt/airflow/dags
  git:
    url: "https://USER:TOKEN@REPO"
    ref: main
    gitSync:
      enabled: true
      resources: {}
      image:
        repository: alpine/git
        tag: latest
        ## values: Always or IfNotPresent
        pullPolicy: Always
      ## the git sync interval in seconds
      ##
      refreshTime: 60
  initContainer:
    ## Fetch the source code when the pods starts
    enabled: true
    ## Image for the init container (any image with git will do)
    image:
      ## docker-airflow image
      repository: k8s.gcr.io/git-sync 
      ## image tag
      tag: v3.1.1
      ## Image pull policy
      ## values: Always or IfNotPresent
      pullPolicy: IfNotPresent
    ## install requirements.txt dependencies automatically
    installRequirements: true
```

其他配置都一样，只不过这里不需要Private key，只需要在url的地方用user, token来访问github就好了

