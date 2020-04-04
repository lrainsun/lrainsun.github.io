---
layout: post
title:  "python中的查找字符串"
date:   2020-03-12 23:00:00 +0800
categories: Python
tags: Python-Language
excerpt: python中的查找字符串
mathjax: true
typora-root-url: ../
---

# match

```python
re.match(pattern, string[, flags])
```

从首字母开始开始匹配，string如果包含pattern子串，则匹配成功，返回Match对象，失败则返回None，若要完全匹配，pattern要以$结尾。

re.match函数只匹配字符串的开始字符，如果开始的字符不符合正则表达式，匹配就会失败，返回None。

# search

```python
re.search(pattern, string[, flags])
```

若string中包含pattern子串，则返回Match对象，否则返回None，注意，如果string中存在多个pattern子串，只返回第一个。

re.search方法匹配整个字符串，直到找到一个匹配的对象，匹配结束没找到匹配值才返回None。

# findall

```python
re.findall(pattern, string[, flags])
```

返回string中所有与pattern相匹配的全部字串，返回形式为数组。

# finditer

```python
re.finditer(pattern, string[, flags])
```

返回string中所有与pattern相匹配的全部字串，返回形式为迭代器。

# 换行

我碰到的问题是，当待查找文本中有换行的时候，用Match是有问题的，比如：

```python
c = """[4872481.415944]  [<ffffffff8d2c1b60>] ? insert_kthread_work+0x40/0x40
[4872601.417557] INFO: task kworker/7:0:14076 blocked for more than 120 seconds.
[4872601.420772] "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message."""
b = re.match("(.*)blocked for more than(.*)seconds",c)
b
print b
None
```

明明c中有我们要找的字符，可是返回的是None，这是因为用了`.`去匹配任意字符，而`.`不能匹配换行符。所以re.match在碰到第一个换行符的时候就返回了

有几个方法可以解决这个问题，我这边只列出两个：

* 用re.search代替re.match

```python
b = re.search("(.*)blocked for more than(.*)seconds",c)
print b
<_sre.SRE_Match object at 0x104308ad0>
```

* 用re.DOTALL

```python
b = re.match("(.*)blocked for more than(.*)seconds",c, re.DOTALL)
print b
<_sre.SRE_Match object at 0x10448dad0>
```

默认情况下，正则表达式中的dot（.），表示所有除了换行的字符，加上re.DOTALL参数后，就是真正的所有字符了，包括换行符（\n）