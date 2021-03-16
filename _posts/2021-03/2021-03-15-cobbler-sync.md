---
layout: post
title:  "cobbler sync"
date:   2021-03-15 23:00:00 +0800
categories: Linux
tags: Linux-Service
excerpt: cobbler sync
mathjax: true
typora-root-url: ../
---

# cobbler sync

由于我们装机的时候，会有多台同时进行pxe配置的情况，每台在add完system之后，都会要去做一下sync的动作

这边cobbler有一个bug，并行的时候会有问题 https://github.com/cobbler/cobbler/issues/1570

所以需要把这个地方改成串行的方式

