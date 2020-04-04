---
layout: post
title:  "ssh private key login失败"
date:   2020-03-24 20:00:00 +0800
categories: Linux
tags: Linux-Tool
excerpt: ssh private key login失败
mathjax: true
typora-root-url: ../
---

# private key Login失败

有台机器一直敲密码，实在敲烦了，就想把public key放进去，用private key登录，可是还是不行

放在/home/ocp/.ssh/authorized_keys了

尝试-v看调试信息

```
ssh -v -i ~/Downloads/rain_keypair.pem ocp@10.225.20.64
debug1: Authentications that can continue: publickey,gssapi-keyex,gssapi-with-mic,password,keyboard-interactive
debug1: Next authentication method: publickey
debug1: Trying private key: /Users/minsu/Downloads/rain_keypair.pem
debug1: Authentications that can continue: publickey,gssapi-keyex,gssapi-with-mic,password,keyboard-interactive
debug1: Next authentication method: keyboard-interactive
```

卡在这里

然后进到主机里，查看日志/var/log/secure

```shell
Mar 24 11:37:43 rain-k8s sshd[2093]: Authentication refused: bad ownership or modes for directory /home/ocp
```

原来是权限问题

/home/ocp权限要设置成755

/home/ocp/.ssh权限要设置成700

/home/ocp/.ssh/authorized_keys要设置成600

owner要是ocp:ocp

改好权限之后，成功登录

