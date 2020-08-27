---
layout: post
title:  "pyadaptivecards"
date:   2020-08-27 23:00:00 +0800
categories: Teams
tags: Teams-Bot
excerpt: pyadaptivecards
mathjax: true
typora-root-url: ../
---

# pyadaptivecards

在调试card的时候，用了pyadaptivecards库，使用的时候发现一些问题

在对Enum做序列化的时候

```python
        for sp in self.simple_properties:
            o = getattr(self, sp, None)

            if getattr(o, 'to_dict', None):
                export[sp] = o.to_dict()
            elif o is not None:
                export[sp] = o
```

由于AbstractOption没有to_dict方法，所以会直接=o，相当于print

```python
from enum import Enum

class AbstractOption(Enum):
    def to_value(self):
        return str(self.name).lower()

    def __str__(self):
        return self.to_value()

    def __repr__(self):
        return self.to_value()
```

而In the Python interactive prompt, if you return a string, it will be displayed with quotes around it, mainly so that you know it's a string.

If you just print the string, it will not be shown with quotes (unless the string has quotes in it).

```python
>>> a='x'
>>> a
'x'
>>> print(a)
x
```

所以序列化之后，我们原本是想要'x'，而实际得到的是x

解决方法是，为AbstractOption定义to_dict()方法