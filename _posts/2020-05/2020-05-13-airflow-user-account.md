---
layout: post
title:  "Airflow用户账户"
date:   2020-05-13 23:01:00 +0800
categories: Airflow
tags: Airflow-Tutorial
excerpt: Airflow用户账户
mathjax: true
typora-root-url: ../E
---

# WEB认证

在安装好airflow，并且启用认证之后，默认是没有用户的。所以需要先创建一个用户

所以根据类似这样，可以用命令行来创建，我们用的是password_auth的认证后端

```shell
airflow users create --username admin --firstname Peter --lastname Parker --role Admin --email spiderman@superhero.org
```

这样启动web认证

```
[webserver]
authenticate = True
auth_backend = airflow.contrib.auth.backends.password_auth
```

# API认证

除此之外，airflow之前讲过还可以支持restful api，而这个api的认证是跟web认证分开来的，也就是说，刚刚创建的用户并不能够用来call api，还是会403的。

可以通过下面这样的方式来设置api的user

```python
$ cd ~/airflow
$ python
Python 2.7.9 (default, Feb 10 2015, 03:28:08)
Type "help", "copyright", "credits" or "license" for more information.
>>> import airflow
>>> from airflow import models, settings
>>> from airflow.contrib.auth.backends.password_auth import PasswordUser
>>> user = PasswordUser(models.User())
>>> user.username = 'new_user_name'
>>> user.email = 'new_user_email@example.com'
>>> user.password = 'set_the_password'
>>> session = settings.Session()
>>> session.add(user)
>>> session.commit()
>>> session.close()
>>> exit()
```

# References

[1] [https://airflow.readthedocs.io/en/latest/security.html?highlight=%20FAB-based#web-authentication](https://airflow.readthedocs.io/en/latest/security.html?highlight= FAB-based#web-authentication)

