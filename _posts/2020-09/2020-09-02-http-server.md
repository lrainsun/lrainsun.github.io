---
layout: post
title:  "http server"
date:   2020-09-02 23:01:00 +0800
categories: Linux
tags: Linux-Service
excerpt: http server
mathjax: true
typora-root-url: ../
---

# http server

最简单的http server

在想要share的文件目录下，执行命令

```shell
[root@ocp-dev-003 airflow-4.0]# python -m SimpleHTTPServer 80
Serving HTTP on 0.0.0.0 port 80 ...
10.79.105.113 - - [02/Sep/2020 08:37:52] "GET /requirements-python3.7.txt HTTP/1.1" 200 -
```

![image-20200902164003867](/../assets/images/image-20200902164003867.png)