---
layout: post
title:  "centos下安装ngrok"
date:   2020-08-21 23:00:00 +0800
categories: Linux
tags: Linux-Service
excerpt: centos下安装ngrok
mathjax: true
typora-root-url: ../
---

# centos下安装ngrok

下载

```shell
wget https://bin.equinox.io/c/4VmDzA7iaHb/ngrok-stable-linux-amd64.zip
```

解压

```shell
unzip ngrok-stable-linux-amd64.zip
```

运行

```shell
./ngrok http localhost:3000
```

