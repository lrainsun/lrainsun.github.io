---
layout: post
title:  "python2 python3 str byte"
date:   2020-05-20 23:00:00 +0800
categories: Python
tags: Python-Language
excerpt: python2 python3 str byte
mathjax: true
typora-root-url: ../
---

# python2 python3 str byte

调试ems代码，从zabbix迁移到alertmanager

原来是python2，现在改成python3

改造之前代码的时候发现这样的提示：

```shell
TypeError: descriptor 'decode' requires a 'bytes' object but received a 'str'
```

这是因为在python2下跟python3下对strings的处理是不一样的

* Python 2 将 strings 处理为原生的 bytes 类型，而不是 unicode

所以python2下，需要encode

```python
auth = 'Basic ' + base64.urlsafe_b64encode("%s:%s" % (self.__username__, self.__password__))

json.dumps(alarm)

request = urllib2.Request(url=self.__url__,
                          data=alarm_content.encode('utf-8'), headers=headers)
```

* Python 3 所有的 strings 均是 unicode 类型。

问题是base64这里，Base64编码仅仅用于面向字节的数据比如字节字符串和字节数组。此外，编码处理的输出结果总是一个字节字符串。 所以编码完成后，还需要再解码成Unicode。

```python
auth = 'Basic %s' % bytes.decode(base64.b64encode(str.encode(auth_str)))
```

# str 与 bytes 之间的类型转换

* str ⇒ bytes：`bytes(s, encoding='utf8')`
* bytes ⇒ str：`str(b, encoding='utf-8')`

此外还可通过编码解码的形式对二者进行转换，

- str 编码成 bytes 格式：`str.encode(s)`
- bytes 格式编码成 str 类型：`bytes.decode(b)`