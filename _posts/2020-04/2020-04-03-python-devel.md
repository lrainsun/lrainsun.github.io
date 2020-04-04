---
layout: post
title:  "pip install error"
date:   2020-04-03 23:00:00 +0800
categories: Python
tags: Python-Language
excerpt: pip install error
mathjax: true
typora-root-url: ../
---

# pip install error

今天在安装psutil的时候，提示

`Python.h: No such file or directory`

这是由于用第三方库使用了c扩展，需要编译，然后又找不到头文件和静态库导致的。编译这些c库需要的依赖库由`python dev`提供，需要安装：

`yum install -y python-devel`

