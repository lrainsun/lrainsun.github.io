---
layout: post
title:  "python for-else"
date:   2021-05-28 23:01:00 +0800
categories: Python
tags: Python-Language
excerpt: python for-else
mathjax: true
typora-root-url: ../
---

# python for-else

孤陋寡闻了。。好歹也算用了python好几年，居然第一次知道，还有for else这种语法呀

那咋用的呢？就是for循环没有遇到任何 break语句的时候，该else子句在循环正常完成时会被执行

感觉是通常在for里面需要处理或者查询某个东西，假如处理好了，或者查询到了，就需要break，言下之意，下面的再也不需要执行了

那如果一直没有break的话，走到else，肯定就是一些针对没有处理，或者没有查找到的一些后续动作了吧

```python
for item in container:
    if search_something(item):
        # Found it!
        process(item)
        break
else:
    # Didn't find anything..
    not_found_in_container()
```

