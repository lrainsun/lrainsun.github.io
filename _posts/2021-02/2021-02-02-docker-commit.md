---
layout: post
title:  "docker commit"
date:   2021-02-02 23:00:00 +0800
categories: Docker
tags: Docker-Service
excerpt: docker commit
mathjax: true
---

# docker commit

我们现在用kolla的horizon image，想要自己自定义一下logo，但是还没有自己build image，为了快速生成image，就先进container把logo改掉，再commit

先起一个干净的container

```
docker run -itd registry-qa.webex.com/ocp/centos-source-horizon:ussuri /bin/bash
```

然后用我们OCP的logo替换原有的logo

因为我们用的source的包，img代码在`/horizon/openstack_dashboard/static/dashboard/img`，运行时在`/var/lib/kolla/venv/lib/python3.6/site-packages/static`

apache中配置

```shell
Alias /static /var/lib/kolla/venv/lib/python3.6/site-packages/static
```

在horizon container起来的时候，会把代码拷贝到这个static目录

```shell
++ /var/lib/kolla/venv/bin/python /var/lib/kolla/venv/bin/manage.py collectstatic --noinput --clear

static files copied to '/var/lib/kolla/venv/lib/python3.6/site-packages/static'.
```

我们修改完文件之后，再出来，把这个container commit成image，如果直接提交`docker commit 4166901f09c4 registry-qa.webex.com/ocp/centos-source-horizon:ussuri.20210128.a`是这样的

```shell
        "Cmd": [
            "/bin/bash"
        ],
```

需要注意的是，我们要把CMD修改成kolla_start，否则跑起来之后会直接退出，因为跑的是/bin/bash

```shell
        "Cmd": [
            "kolla_start"
        ],
```

`docker commit --change='CMD ["kolla_start"]' 4166901f09c4 registry-qa.webex.com/ocp/centos-source-horizon:ussuri.20210128.a`

