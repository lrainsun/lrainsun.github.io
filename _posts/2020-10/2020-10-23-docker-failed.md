---
layout: post
title:  "docker stop container失败"
date:   2020-10-23 23:00:00 +0800
categories: Docker
tags: Docker-Service
excerpt: docker stop container失败
mathjax: true
---

# docker stop container失败

今天碰到在stop一个container的时候，失败

```shell
[root@prometheus ~]# docker stop grafana
Error response from daemon: Cannot stop container grafana: Cannot kill container 07ada1ae2e7dc24c57a92b49804867052af08a256b4e3820974b203ebf678927: rpc error: code = 14 desc = grpc: the connection is unavailable
```

强制删除也不行

```shell
[root@prometheus ~]# docker rm -f grafana
Error response from daemon: Could not kill running container 07ada1ae2e7dc24c57a92b49804867052af08a256b4e3820974b203ebf678927, cannot remove - Cannot kill container 07ada1ae2e7dc24c57a92b49804867052af08a256b4e3820974b203ebf678927: rpc error: code = 14 desc = grpc: the connection is unavailable
```

尝试进到容器里

```shell
[root@prometheus ~]# docker exec -it grafana bash
rpc error: code = 14 desc = grpc: the connection is unavailable
```

然后发现该机器上所有其他的容器也都有问题

debug模式看一下

```shell
[root@prometheus ~]# docker-containerd -l unix:///var/run/docker/libcontainerd/docker-containerd.sock --metrics-interval=0 --start-timeout 2m --state-dir /var/run/docker/libcontainerd/containerd --shim docker-containerd-shim --runtime docker-runc --debug
WARN[0000] containerd: low RLIMIT_NOFILE changing to max  current=1024 max=4096
DEBU[0000] containerd: grpc api on /var/run/docker/libcontainerd/docker-containerd.sock
DEBU[0000] containerd: read past events                  count=78
ERRO[0000] containerd: notify OOM events                 error=open /proc/18972/cgroup: no such file or directory
DEBU[0000] containerd: container restored                id=07ada1ae2e7dc24c57a92b49804867052af08a256b4e3820974b203ebf678927
DEBU[0000] containerd: container restored                id=75026a060eed1f2fb0d8ff0b29c7a268ae6f5bc4c1c87f32a7c19c7ca374ff20
DEBU[0000] containerd: container restored                id=8b2b3957ab7ecb40ccc5f6cd9c9ecd4a54848a6128d1d114b7e674782e6f6b23
DEBU[0000] containerd: container restored                id=bfaab93e15c9bf0d4cf800206fe2357eb6c663dd71bd1a1579d9423640b54050
DEBU[0000] containerd: container restored                id=df18ab29872d991b2ac2cf2fe4c67708ea18a66cff56f6331a5a4a77a72b2511
DEBU[0000] containerd: container restored                id=ebbea138737ee2495e74224aac882948007c7a0a0809912928a689ada7e04d5d
DEBU[0000] containerd: supervisor running                cpus=16 memory=64264 runtime=docker-runc runtimeArgs=[] stateDir=/var/run/docker/libcontainerd/containerd
DEBU[0000] containerd: process exited                    id=07ada1ae2e7dc24c57a92b49804867052af08a256b4e3820974b203ebf678927 pid=init status=1 systemPid=18972
ERRO[0000] containerd: deleting container                error=exit status 1: "container 07ada1ae2e7dc24c57a92b49804867052af08a256b4e3820974b203ebf678927 does not exist\none or more of the container deletions failed\n"
```

感觉是docker服务出了问题，重启以后得到解决

```shell
[root@prometheus ~]# systemctl restart docker
```

