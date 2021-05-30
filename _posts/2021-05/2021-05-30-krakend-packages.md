---
layout: post
title:  "Krakend"
date:   2021-05-30 23:00:00 +0800
categories: Gateway
tags: API-Gateway
excerpt: Krakend packages
mathjax: true
typora-root-url: ../
---

# Krakend packages

krakend有几个重要原则：

- Reactive is key (yes, it is very very important)
- Failing fast is better than succeeding slow (say it one more time!)
- The simpler, the better
- Everything is pluggable
- Each request must be processed in its request-scoped context

# krakend内部状态

## building状态

服务启动之后，可以开始接受traffic的状态：

1. 解析配置以修复系统行为
2. 准备middlewares
3. 构造pipes （pipe是一个function -接收request message，process，生成response message或error，krakend router把pipe绑定到一个tranport层，比如http, grpc）

当building状态完成之后，krakend就不需要再calculate任何route或者lookup相关联的handler function，因为这些mapping都提前被放在内存里了

## working状态

working状态就是系统准备好，可以开始处理request的状态。一个request过来的时候，router以及有了关于handler function的mapping，所以直接trigger pipe的执行。proxy用来执行pipe中manipulates, aggregates, 或者其他数据处理的过程。因为Pipe已经提前load到内存了，所以KrakenD不会因为endpoint的数量或用户请求的URI的基数来影响性能。

# 重要的packages

1. config：定义service
2. router：set up暴露给clients的endpoint
3. proxy：添加所需要的middlewares和components，以进一步处理request的发送过程，以及接受backends的response，而且管理了跟backend的连接

除此之外，framework还包括了一些helpers，adapters（encoding, logging, service discovery）

proxy是大部分krakend components和features存在的地方，定义了两个重要的接口：

* Proxy is a function that converts a given context and request into a response.
* Middleware is a function that accepts one or more proxies and returns a single proxy wrapping them.

这一层的目的是把从外层（route）收到的request转成一个或者多个requests给backend services，然后process responses并且返回single reponse

一部分comminity的middleware和components

https://github.com/devopsfaith/krakend-contrib