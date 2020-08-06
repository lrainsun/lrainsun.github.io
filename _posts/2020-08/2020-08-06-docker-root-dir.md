---
layout: post
title:  "docker修改root dir"
date:   2020-08-06 23:00:00 +0800
categories: Docker
tags: Docker-Volume
excerpt: docker修改root dir
mathjax: true
typora-root-url: ../
---

# docker修改root dir

在我们的标准image里，/var目录只有9g，而docker默认的root dir在/var/lib/docker下，所以经常会满

```shell
 Docker Root Dir: /var/lib/docker
 
 /dev/vda3       9.0G  9.0G   20K 100% /var
 
 [root@bmaas ~]# sudo du -hs /var/lib/docker/
12G	/var/lib/docker/

[root@bmaas ~]# docker system df
TYPE                TOTAL               ACTIVE              SIZE                RECLAIMABLE
Images              12                  11                  2.598GB             999.1MB (38%)
Containers          12                  12                  38.83kB             0B (0%)
Local Volumes       8                   8                   1.272GB             0B (0%)
Build Cache         0                   0                   0B                  0B
```

## docker system prune

docker system prune 命令可以用于清理磁盘，删除关闭的容器、无用的数据卷和网络，以及dangling镜像(即无tag的镜像)

docker system prune -a 命令清理得更加彻底，可以将没有容器使用Docker镜像都删掉

## 迁移root dir

最好的方法还是迁移root dir

把docker先停掉

```shell
service docker stop
```

然后创建新的docker目录，并同步数据

```shell
mkdir -p /home/ocpadmin/docker
rsync -avz rsync -avz /var/lib/docker /home/ocpadmin/docker/
```

或者

```shell
cp rsync -avz /var/lib/docker /home/ocpadmin/docker/ -rf
```

配置root dir

因为我们用的kolla部署，所以在/etc/systemd/system/docker.service.d/下有一个Kolla.conf，直接编辑这个文件就可以。如果没有可以新建一个文件，命名随意

```shell
[root@ci81hf1cmp001 ~]# cat /etc/systemd/system/docker.service.d/kolla.conf
[Service]
MountFlags=shared
ExecStart=
ExecStart=/usr/bin/dockerd --log-opt max-file=5 --log-opt max-size=50m --graph=/home/ocpadmin/docker
```

重新加载docker

```shell
systemctl daemon-reload
systemctl restart docker
systemctl enable docker
```

这时docker info查看，root dir已经改过来了

```shell
Docker Root Dir: /home/ocpadmin/docker
```

检查一下容器和镜像是否都正常

```shell
[root@ci81hf1cmp001 ~]# docker images
REPOSITORY                                           TAG                 IMAGE ID            CREATED             SIZE
registry-qa.webex.com/ocp/distributed_ocp_exporter   v1.4                6f22414456a7        6 weeks ago         255MB
registry-qa.webex.com/ocp/distributed_ocp_exporter   <none>              823229a7bb20        6 weeks ago         763MB
registry-qa.webex.com/ocp/libvirt-exporter           v1.4                2f1c1af30120        6 weeks ago         16MB
registry-qa.webex.com/ocp/rabbitmq-exporter          v1.3                8ccc7c75b0e5        10 months ago       13.4MB
registry-qa.webex.com/ocp/rabbitmq-exporter          v1.4                8ccc7c75b0e5        10 months ago       13.4MB
registry-qa.webex.com/ocp/memcached-exporter         v1.3                2189a810f27b        11 months ago       16.2MB
registry-qa.webex.com/ocp/memcached-exporter         v1.4                2189a810f27b        11 months ago       16.2MB
registry-qa.webex.com/ocp/mysqld-exporter            v1.3                09f404b8b594        11 months ago       17.5MB
registry-qa.webex.com/ocp/mysqld-exporter            v1.4                09f404b8b594        11 months ago       17.5MB
registry-qa.webex.com/ocp/node-exporter              v1.3                7583142249dc        11 months ago       22.9MB
registry-qa.webex.com/ocp/node-exporter              v1.4                7583142249dc        11 months ago       22.9MB
registry-qa.webex.com/ocp/haproxy-exporter           v1.3                9f0642e9c37c        11 months ago       15.3MB
registry-qa.webex.com/ocp/haproxy-exporter           v1.4                9f0642e9c37c        11 months ago       15.3MB
[root@ci81hf1cmp001 ~]# docker ps
CONTAINER ID        IMAGE                                                     COMMAND                  CREATED             STATUS              PORTS               NAMES
4f25e98b5c0d        registry-qa.webex.com/ocp/distributed_ocp_exporter:v1.4   "/bin/sh -c 'python3…"   3 days ago          Up 11 hours                             prometheus_distributed_ocp_exporter
2780f31d66a2        registry-qa.webex.com/ocp/libvirt-exporter:v1.4           "./libvirt_exporter …"   3 days ago          Up 11 hours                             prometheus_libvirt_exporter
811f502706d5        registry-qa.webex.com/ocp/haproxy-exporter:v1.4           "/bin/haproxy_export…"   5 days ago          Up 11 hours                             prometheus_haproxy_exporter
```

都正常说明迁移成功了



