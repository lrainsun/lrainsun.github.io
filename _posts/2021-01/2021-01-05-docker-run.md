---
layout: post
title:  "docker run"
date:   2021-01-05 23:00:00 +0800
categories: Docker
tags: Docker-Service
excerpt: docker run
mathjax: true
---

# docker run

我们做了一个image，可以快速的用docker run跑一下看看里面的内容是否符合预期

由于entrypoint可能会要求配置文件之类的，可以修改entrypoint来快速跑一下

```shell
docker run
docker run -it --entrypoint /bin/bash registry-qa.webex.com/ocp/zero-touch-service:v1.0.20
```

如果在这个image里做了修改想要提交，因为我们docker run的时候修改了entrypoint，可以在commit的时候修改回来

```shell
docker commit --change='ENTRYPOINT ["zero-touch"]' 8f24396953b5 registry-qa.webex.com/ocp/zero-touch-service:v1.0.21
```

