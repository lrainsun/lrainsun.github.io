---
layout: post
title:  "高性能NoSQL"
date:   2021-02-14 23:02:00 +0800
categories: Architecture
tags: Architecture
excerpt: 高性能NoSQL
mathjax: true
typora-root-url: ../
---

# 高性能NoSQL

关系数据库虽然应用广泛，但是还是存在如下缺点：

* 关系数据库存储的是行记录，无法存储数据结构
* 关系数据库的schema扩展很不方便
* 关系数据库在大数据场景下I/O较高
* 关系数据库的全文搜索功能比较弱

针对上述问题，分别诞生了不同的NoSQL解决方案，这些方案与关系数据库相比，在某些应用场景下表现更好。但世上没有免费的午餐，NoSQL方案带来的优势，本质上是牺

牲ACID中的某个或者某几个特性，因此我们不能盲目地迷信NoSQL是银弹，而应该将NoSQL作为SQL的一个有力补充，NoSQL != No SQL，而是NoSQL = Not Only SQL。

常见的NoSQL方案分为4类。

* K-V存储：解决关系数据库无法存储数据结构的问题，以Redis为代表。
* 文档数据库：解决关系数据库强schema约束的问题，以MongoDB为代表。
* 列式数据库：解决关系数据库大数据场景下的I/O问题，以HBase为代表。
* 全文搜索引擎：解决关系数据库的全文搜索性能问题，以Elasticsearch为代表。