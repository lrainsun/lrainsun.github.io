---
layout: post
title:  "python2的python3的print"
date:   2020-03-11 23:00:00 +0800
categories: Python
tags: Python-Language
excerpt: python2的python3的print
mathjax: true
typora-root-url: ../
---

最近写exporter用的是python3，print的时候总是会出错，发现python2跟python3中print是有些不一样的

# 函数

* python2中的print不是个函数
  *  `print "hi"` 和 `print("hi")`都可以
* python3中的print是一个函数
  * 只能够`print("hi")`

# 帮助

python2里的print因为不是函数，所以不能够使用help

而python3的print因为是函数，所以可以用help查看文档

```python
>>> help(print)
Help on built-in function print in module builtins:

print(...)
    print(value, ..., sep=' ', end='\n', file=sys.stdout, flush=False)

    Prints the values to a stream, or to sys.stdout by default.
    Optional keyword arguments:
    file:  a file-like object (stream); defaults to the current sys.stdout.
    sep:   string inserted between values, default a space.
    end:   string appended after the last value, default a newline.
```

# 重定向

python2里

```python
>>> with open('test.txt', 'w') as f:
...     print >> f, "hi, rain"
...
>>>
[root@rain-working-vm ~]# cat test.txt
hi, rain
```

python3里

```python
>>> with open('test.txt', 'w') as f:
...     print("hi, rain", file = f)
...
>>>
sh-4.2# cat test.txt
hi, rain
```

# sep & end

sep:   string inserted between values, default a space.

```python
>>> list = [1, 2, 3, 4, 5]
>>> print(*list, sep = "/")
1/2/3/4/5
```

end:   string appended after the last value, default a newline.

```python
>>> print(*list, end = "&&&\n")
1 2 3 4 5&&&
>>> print(*list, sep = "*", end = "&&&\n")
1*2*3*4*5&&&
```

