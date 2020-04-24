---
layout: post
title:  "Airflow -- kubernetes"
date:   2020-04-24 23:00:00 +0800
categories: Airflow
tags: Airflow-Tutorial
excerpt: Airflow -- kubernetes
mathjax: true
typora-root-url: ../
---

#  Airflow -- kubernetes

只是纯理论，并没有什么实践。前面学习过airflow里有很多很多operator，比如python的呀（光python里面就还有各种类型，Branch的，ShortCircuit啥的），有Bash的，等等。

文章[[1]([https://medium.com/bluecore-engineering/were-all-using-airflow-wrong-and-how-to-fix-it-a56f14cb0753](https://medium.com/bluecore-engineering/were-all-using-airflow-wrong-and-how-to-fix-it-a56f14cb0753))] 提到：

>  Airflow Operators, instead of simply orchestrating work to be executed, actually implement some of the functional work themselves. This means that Airflow Operators inherently combine orchestration bugs with execution bugs.

所以看起来我们最好使用kubernetes operator，把task变成pod来跑比较靠谱

嗯，今天就简单点吧，因为今天大部分时间在搞DAG

# References

[1] [https://medium.com/bluecore-engineering/were-all-using-airflow-wrong-and-how-to-fix-it-a56f14cb0753](https://medium.com/bluecore-engineering/were-all-using-airflow-wrong-and-how-to-fix-it-a56f14cb0753)

[2] [https://www.techatbloomberg.com/blog/airflow-on-kubernetes/](https://www.techatbloomberg.com/blog/airflow-on-kubernetes/)