---
layout: post
title:  "airflow git-sync导致pod restart"
date:   2020-11-27 23:00:00 +0800
categories: Airflow
tags: Airflow-Tutorial
excerpt: airflow git-sync导致pod restart
mathjax: true
typora-root-url: ../
---

# airflow git-sync导致pod restart

一直有个问题，我们的airflow的pod经常会restart

```shell
MINSU-M-M1RW:ansible minsu$ kubectl --kubeconfig ~/Documents/k8s/oke-dev-ci02qa-a.yaml -n iaas-airflow get pod --field-selector=status.phase=Running
NAME                                      READY   STATUS    RESTARTS   AGE
iaas-airflow-postgresql-0                 1/1     Running   2          23d
iaas-airflow-scheduler-74d48464c8-fb2dh   2/2     Running   160        5d12h
iaas-airflow-web-57768cb64c-q5fgf         2/2     Running   38         2d22h
```

airflow1.10.12的helm chart把scheduler跟web分开成两个pod了，而我们的dags是从github上面取的，dags用的又是共享存储（nfs），也就是说，scheduler跟web都会要去git clone，而两者同时去sync的时候就会发生问题

```shell
MINSU-M-M1RW:ansible minsu$ kubectl --kubeconfig ~/Documents/k8s/oke-dev-ci02qa-a.yaml -n iaas-airflow logs -f --tail 20 iaas-airflow-scheduler-74d48464c8-fb2dh -c git-sync -p
Cloning into '/dags'...
From https://sdpgit.webex.com/iaas_cm/ocp/ocp_dags
 * branch            dags       -> FETCH_HEAD
fatal: Unable to create '/dags/.git/index.lock': File exists.
```

workaround的是，修改git-sync的脚本

```shell
kubectl --kubeconfig ~/Documents/k8s/oke-dev-ci02qa-a.yaml -n iaas-airflow edit configmap iaas-airflow-scripts-git
  git-sync.sh: |
    #!/bin/sh -e
    [ -f /dags/.git/index.lock ] && exit 0
    REPO=$1
    REF=$2
    DIR=$3
    REPO_HOST=$4
    REPO_PORT=$5
    PRIVATE_KEY=$6
    SYNC_TIME=$7

    mkdir -p ~/.ssh/
    if [ -d "$DIR" ]; then
      rm -rf $( find $DIR -mindepth 1 )
    fi
    [ -f /dags/.git/index.lock ] && exit 0
    git clone $REPO -b $REF $DIR

    # to break the infinite loop when we receive SIGTERM
    trap "exit 0" SIGTERM

    cd $DIR
    while true; do
      git fetch origin $REF;
      git reset --hard origin/$REF;
      git clean -fd;
      date;
      sleep $SYNC_TIME;
    done
```

在sync一开始以及clone之前，就去判断这个文件，如果文件存在，说明有其他人正在sync，那就直接退出

```shell
  git-sync.sh: |
    #!/bin/sh -e
    [ -f /dags/.git/index.lock ] && exit 0
    REPO=$1
    REF=$2
    DIR=$3
    REPO_HOST=$4
    REPO_PORT=$5
    PRIVATE_KEY=$6
    SYNC_TIME=$7

    mkdir -p ~/.ssh/
    if [ -d "$DIR" ]; then
      rm -rf $( find $DIR -mindepth 1 )
    fi
    [ -f /dags/.git/index.lock ] && exit 0
    git clone $REPO -b $REF $DIR

    # to break the infinite loop when we receive SIGTERM
    trap "exit 0" SIGTERM

    cd $DIR
    while true; do
      git fetch origin $REF;
      git reset --hard origin/$REF;
      git clean -fd;
      date;
      sleep $SYNC_TIME;
    done
```

修改之后，发现还是会偶尔有restart，不一样的错误

```shell
MINSU-M-M1RW:ocp_dags minsu$ kubectl --kubeconfig ~/Documents/k8s/oke-dev-ci02qa-a.yaml -n iaas-airflow logs -f --tail 20 iaas-airflow-web-57768cb64c-7n56j -c git-sync -p
HEAD is now at 01066e7 check rabbitmq patch
Fri Nov 27 01:08:54 UTC 2020
From https://sdpgit.webex.com/iaas_cm/ocp/ocp_dags
branch dags -> FETCH_HEAD HEAD is now at 01066e7 check rabbitmq patch Fri Nov 27 01:09:54 UTC 2020 From https://sdpgit.webex.com/iaas_cm/ocp/ocp_dags
branch dags -> FETCH_HEAD HEAD is now at 01066e7 check rabbitmq patch Removing pycache/ Fri Nov 27 01:10:54 UTC 2020 From https://sdpgit.webex.com/iaas_cm/ocp/ocp_dags
branch dags -> FETCH_HEAD HEAD is now at 01066e7 check rabbitmq patch Fri Nov 27 01:11:55 UTC 2020 From https://sdpgit.webex.com/iaas_cm/ocp/ocp_dags
branch dags -> FETCH_HEAD HEAD is now at 01066e7 check rabbitmq patch warning: could not open directory 'pycache/': No such file or directory fatal: Cannot lstat 'pycache/': No such file or directory
```

我们手工到pod里去试一下

```shell
[root@oke-dev-ci02qa-a-worker-1 dags-new]# git fetch origin dags
From https://sdpgit.webex.com/iaas_cm/ocp/ocp_dags
 * branch            dags       -> FETCH_HEAD
[root@oke-dev-ci02qa-a-worker-1 dags-new]# git reset --hard origin/dags
HEAD is now at 01066e7 check rabbitmq patch
[root@oke-dev-ci02qa-a-worker-1 dags-new]# git clean -fd
Removing __pycache__/
[root@oke-dev-ci02qa-a-worker-1 dags-new]# date
Fri Nov 27 01:27:49 GMT 2020
```

发现出错是因为这句`git clean -fd`，git clean是为了删除一些没有git add 的文件；*git clean* 参数 -n 显示将要删除的文件和 目录 -f 删除文件，-df 删除文件和目录。在我们的场景里，这句其实是可以不需要的，所以我先注释掉了看一下效果