---
layout: post
title:  "docker image build"
date:   2020-06-05 23:00:00 +0800
categories: Docker
tags: Docker-Image
excerpt: docker image build
mathjax: true
typora-root-url: ../
---

# docker image build

今天在build docker image的时候，想要拷贝一个文件夹到image里

```shell
[root@airflow airflow]# ls -al | grep ansible
drwxrwxrwx. 12 root root   4096 Jun  4 13:44 ocp_ansible_module-2.7.2004.0
```

```shell
COPY ocp_ansible_module-2.7.2004.0 "${AIRFLOW_HOME}/ocp_ansible_module-2.7.2004.0"
```

但是怎么build都拷贝不进去，最后发现根目录有一个`.dockerignore`文件

## .dockerignore

`.dockerignore` 文件的作用类似于 git 工程中的 `.gitignore` 。不同的是 `.dockerignore` 应用于 docker 镜像的构建，它存在于 docker 构建上下文的根目录，用来排除不需要上传到 docker 服务端的文件或目录。

![image-20200605211309609](/../assets/images/image-20200605211309609.png)

如果两个匹配语法规则有包含或者重叠关系，那么以后面的匹配规则为准，比如：

```shell
*.md
!README*.md
README-secret.md
```

这么写的意思是将根路径下所有以 `.md` 结尾的文件排除，以 `README` 开头 `.md` 结尾的文件保留，但是 `README-secret.md` 文件排除。

```shell
*.md
README-secret.md
!README*.md
```

这么写的意思是将根路径下所有以 .md 结尾和名称为 README-secret.md 的文件排除，但所有以 README 开头 .md 结尾的文件保留。这样的话 README-secret.md 依旧会被保留，并不会被排除，因为 README-secret.md 符合 !README*.md 规则。

## 应用

所以在.dockerignore文件中增加以下配置，就可以把文件夹编译进去了

```shell
!ocp_ansible_module-2.7.2004.0
```

可是在跑playbook的时候发现报错，发现`ocp_ansible_module-2.7.2004.0/files/patch/usr/lib`明明有却又没被编译进去

果然又是这个.dockerignore干的

```shell
# Exclude python generated files
**/lib/
**/lib64/
```

所以，把这两条加进去

```shell
!ocp_ansible_module-2.7.2004.0/files/patch/usr/lib
!ocp_ansible_module-2.7.2004.0/templates/usr/lib
```

