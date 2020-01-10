---
layout: post
title:  "TFTP启动失败"
date:   2020-10-10 23:00:00 +0800
categories: Linux
tags: Linux-Service
excerpt: TFTP启动失败
mathjax: true
typora-root-url: ../
---

# TFTP

前两天在通过kolla装standalone  ironic的时候，ironic_pxe总是起不来，看了一下，跑的command也很简单，就是启动tftp服务，并没有什么特别的

```shell
/usr/sbin/in.tftpd --verbose --foreground --user root --address 0.0.0.0:69 --map-file /map-file /tftpboot
```

进到container里面去跑，依然还是跑不起来。google了好久。。drive me crazy...

以为是不是我系统哪个地方配置有问题，于是尝试在container外面的vm上启动tftp服务。安装tftp，依赖tftp.socket，一开始还以为是不是tftp.socket在container里面没启动？

```shell
service tftp start

[root@rain-standalone-ironic ~]# cat /usr/lib/systemd/system/tftp.service
[Unit]
Description=Tftp Server
Requires=tftp.socket
Documentation=man:in.tftpd

[Service]
ExecStart=/usr/sbin/in.tftpd -s /var/lib/tftpboot
StandardInput=socket

[Install]
Also=tftp.socket
```

最后直接在vm跑命令的时候，查看系统日志/var/log/message，发现这样一条错误日志：

```shell
Jan 10 08:23:08 localhost in.tftpd[17225]: cannot open IPv6 socket, disable IPv6: Address family not supported by protocol
Jan 10 08:23:08 localhost in.tftpd[17225]: Cannot set nonblock flag on socket: Bad file descriptor
```

> ```
> It turns out that the code has support of continuing without ipv6 if the ipv6
> fails but ipv4 is opened succesfully, but it even does some stuff with the ipv6 socket, even when it failed to open, resulting in this error and the daemon quitting.
> The reason that the ipv6 socket fails to open is that I'm running a custom kernel with ipv6 disabled
> ```

在我的环境里没有启用ipv6，所以会有这样的问题，一个workaround是增加"-4"强制tftp使用ipv4。最终把ironic_pxe的启动command改成下面这样，就可以成功启动啦。妈呀，折腾了我好久

```shell
/usr/sbin/in.tftpd --verbose -4 --foreground --user root --address 0.0.0.0:69 --map-file /map-file /tftpboot

CONTAINER ID        IMAGE                                        COMMAND                  CREATED             STATUS              PORTS               NAMES
f6bf144a8352        kolla/centos-binary-ironic-pxe:train         "dumb-init --single-…"   5 hours ago         Up 5 hours                              ironic_pxe
```

[1] [https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=544089](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=544089)

[2] [https://forums.fogproject.org/topic/13278/tftp-won-t-start-at-the-installation](https://forums.fogproject.org/topic/13278/tftp-won-t-start-at-the-installation)