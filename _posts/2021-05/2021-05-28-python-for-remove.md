---
layout: post
title:  "python for remove"
date:   2021-05-28 23:02:00 +0800
categories: Python
tags: Python-Language
excerpt: python for remove
mathjax: true
typora-root-url: ../
---

# python for remove

这也是吃了亏的，在for循环，去remove list里的元素，是会有问题的，看下面就明白了

```python
>>> x = [1,2,3,4,5,6,7,8,9,10]
>>> for i in x:
...     print(i)
...
1
2
3
4
5
6
7
8
9
10
```

在看这个

```python
>>> for i in x:
...     print(i)
...     if i % 3 == 0:
...             x.remove(i)
...
1
2
3
5
6
8
9
>>> x
[1, 2, 4, 5, 7, 8, 10]
```

实际4，7，10被跳过了，并不是list中所有的元素都被处理到了

虽然在这个例子里不会有影响，但是实际是有问题的

