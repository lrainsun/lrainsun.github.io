---
layout: post
title:  "haproxy session limit"
date:   2020-02-14 22:00:00 +0800
categories: Linux
tags: Linux-Service
excerpt: haproxy session limit
mathjax: true
typora-root-url: ../
---

# Session Limit

在我们的环境中，有这样一条alert rule

```yaml
(haproxy_backend_current_sessions
 / haproxy_backend_limit_sessions * 100) > 90
```

有个环境一直在告警，就想看看是怎么回事

![image-20200214143521855](/../assets/images/image-20200214143521855.png)

backend session limit是1639，而current session一直维持在1886，不会有问题吗？

# Backend Session Limit

查了一下haproxy backend session limit

The Session Limit in the backend row refers to the fullconn parameter of that backend. The fullconn is about setting dynamic maxconn on backend servers. You can read more about it in the docs
[http://cbonte.github.io/haproxy-dconv/1.7/configuration.html#4-fullconn 1.4k](http://cbonte.github.io/haproxy-dconv/1.7/configuration.html#4-fullconn)

You need to consider the “fullconn” parameter if you have set up “minconn” in server lines (to use dynamic maxconn), otherwise you can ignore it.

原来这个数值跟backend server是否设置minconn有关，如果backend server设置了这个值，那么backend session limit才是有效的。而实际上我们并没有设置这个minconn，所以其实可以忽略这个alert。

# Frontend Session Limit

这个frontend session limit关系着可以接受多少request

The *Limit* column shows the most simultaneous sessions that are allowed, as defined by the `maxconn` setting in the `frontend`. That particular frontend will stop accepting new connections when this limit is reached. If `maxconn` is not set, then *Limit* is the same as the `maxconn` value in the `global` section of your configuration. If that isn’t set, the value is based on your system.

maxconn的值我们是设置了的，也就是16384，当超过这个数值时，request就会有问题。alert可以对frontend current session是否超过limit进行监控。