---

layout: post
title:  "python中的函数参数"
date:   2020-12-03 23:00:00 +0800
categories: Python
tags: Python-Language
excerpt: python中的函数参数
mathjax: true
typora-root-url: ../
---

# python中的函数参数

Python的函数具有非常灵活的参数形态，既可以实现简单的调用，又可以传入非常复杂的参数。

## 位置参数

位置参数就是必须按照顺序定义的参数，调用函数时根据函数定义的参数位置来传递参数

```python
def func(one, two, three)
```

那么调用的时候

```python
func(1, 2, 3)
```

one就等于1，two等于2，three等于3

## 默认参数

上面的函数，我们还可以定义默认值

```python
def func(one, two, three=3)
```

那么就可以这样调用，默认three就是3

```python
func(1, 2)
```

## 可变参数

可变参数就是传入的参数个数是可变的，可以是0个，1个，2个，……很多个。就是可以一次给函数传很多的参数

> *args是可变参数，args接收的是一个tuple；

## 关键字参数

关键字参数允许你传入0个或任意个含参数名的参数，这些关键字参数在函数内部自动组装为一个dict。在调用函数时，可以只传入必选参数。可以扩展函数的功能。

> **kw是关键字参数，kw接收的是一个dict。

