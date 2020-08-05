---
layout: post
title:  "获取airflow运行的dag run状态"
date:   2020-08-05 23:00:00 +0800
categories: Airflow
tags: Airflow-Tutorial
excerpt: 获取airflow运行的dag run状态
mathjax: true
typora-root-url: ../
---

# 获取airflow运行的dag run状态

当使用airflow的时候，有时候我们会想要知道task的运行状态和结果，除了在界面上看到之外

可以使用几下几种方法：

1. dag run的时候主动上报状态： 
	on_failure_callback
	on_retry_callback
	on_success_callback
2. api可以获取到某一个dag run的某一个task的具体执行状态
   但是通过api获取不到某dag下的有哪些task，以及task之间的依赖关系
3. 通过cli可以获取到所有的信息，包括所有task以及依赖关系，包括re-run, clean
   但是cli需要ssh到airflow container才能执行

