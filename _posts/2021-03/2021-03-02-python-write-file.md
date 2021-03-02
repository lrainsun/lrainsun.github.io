---
layout: post
title:  "python写文件没有结束符"
date:   2021-03-02 23:00:00 +0800
categories: Python
tags: Python-Language
excerpt: python写文件没有结束符
mathjax: true
typora-root-url: ../
---

# python写文件没有结束符

今天调试的时候，发现一个问题，从vault上面download了Privatekey，然后Git clone，一直失败

```shell
airflow@vmwareaddesxihostfor1216616d72924ffa9c09f7add504188breconcilead:/opt/airflow/dags$ git clone git@sdpgit.webex.com:webex-iaas/compute/esx_operations.git /tmp/compute/esx_operations
Cloning into '/tmp/compute/esx_operations'...
Load key "/home/airflow/.ssh/id_rsa": invalid format
git@sdpgit.webex.com: Permission denied (publickey).
fatal: Could not read from remote repository.
```

但是检查了Privatekey明明就是对的，更奇怪的是，我vim进去看了一下，没有修改任何内容，再保存一下退出，竟然就可以了，可是我明明啥都没改呀

写文件代码是这样的

```python
        with open(file, 'wb') as f:
            if isinstance(content, str):
                f.write(str.encode(content))
            else:
                f.write(content)
```

重新再生成文件，再vim进去，发现有个奇怪的符号

![img](/../assets/images/54178107.png)

Google了一下，'noeol' 就是 'no end-of-line', 即“没有行末结束符”

我们来看一下这个Privatekey

```
airflow@vmwareaddesxihostfor1216616d72924ffa9c09f7add504188breconcilead:/opt/airflow$ cat -A /home/airflow/.ssh/id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----$
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn$
NhAAAAAwEAAQAAAQEAytR3cF/2Yfjv5DHWVVHpG54yE+xayYr5zqYp2bPknnEFcBwJfaLe$
nt0fXPzEI8xXDNRKvJqrn0KbCBHv8ykVSeKknw2XiiQVVUaxQkWs+A3DJLR9JhJYZ0iGwN$
8dmiAPWPI4NidMAknP1ueHRBuqEoERGYHDkaACyU3OjImHu4hCyzNJLDdvkOltoP/zD4rF$
……
2DVeQZhUhlNrJ9BDKqUxTgGa3rJXBTIA1Mz21qiHqgPi/hAAAAgQDe3ez6SCQegQJ5KVa3$
c+uZDkaNeTkdhVZyY3WRR3FrTgIax4GACkYQ2ypdMzkdh60/ZNIqrBrKZJBcOn0QTnelP8$
73oHLJhapYW1GaLMAfTdPTJpYCoojbdMsAO/3zqp18FPcLwFMkAZD5V/N41wEJEedI3nvM$
9FSkXq9w9ZHlHQAAABhtYXRkYXZpZEBNQVREQVZJRC1NLVQySFcBAgME$
-----END OPENSSH PRIVATE KEY-----
```

每一行末尾都会有一个结束符$，而在最后一行却没有，重新保存一下之后，就会发现这个结束符被添加上了。原来如此。

好像这个问题在一般文件上都没啥问题，作为git clone的privatekey来使用的时候才有这样的问题，暂时没有想到python里比较好的处理方法，所以在文件生成后直接做了个简单的workaround

```python
os.system('echo "" >> /home/airflow/.ssh/id_rsa')
```

