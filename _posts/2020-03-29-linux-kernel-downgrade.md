---
layout: post
title:  "Linux Kernel Downgrade"
date:   2020-03-29 23:00:00 +0800
categories: Openstack
tags: Linux-Kernel
excerpt: Linux Kernel Downgrade
mathjax: true
---

# Linux Kernel Downgrade

我们当前版本是`3.10.0-957.12.2.el7.x86_64`，想要downgrade到 `3.10.0-1062.9.1.el7.x86_64`

```shell
[root@ci81hf1cmp009 ~]# uname -a
Linux ci81hf1cmp009.qa.webex.com 3.10.0-957.12.2.el7.x86_64 #1 SMP Tue May 14 21:24:32 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
```

在GRUN2文件中列出kernel entires，前提是这里得有

```shell
[root@ci81hf1cmp009 ~]# awk -F\' '$1=="menuentry " {print $2}' /etc/grub2.cfg
CentOS Linux (3.10.0-957.12.2.el7.x86_64) 7 (Core)
CentOS Linux (3.10.0-1062.12.1.el7.x86_64) 7 (Core)
CentOS Linux (3.10.0-1062.9.1.el7.x86_64) 7 (Core)
CentOS Linux (0-rescue-ca345ecbebb047328b4bb3d19a66ecbc) 7 (Core)
```

`3.10.0-957.12.2.el7.x86_64`在第三行，entry是2

```shell
[root@ci81hf1cmp009 ~]# grub2-set-default 2
```

然后重启

```shell
[root@ci81hf1cmp009 ~]# shutdown -r now
```

起来后就切换成功了

```shell
[ocpadmin@ci81hf1cmp009 ~]$ uname -a
Linux ci81hf1cmp009.qa.webex.com 3.10.0-1062.9.1.el7.x86_64 #1 SMP Fri Dec 6 15:49:49 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
```

