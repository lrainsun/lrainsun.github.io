---
layout: post
title:  "在容器内使用psutil"
date:   2020-04-04 23:01:00 +0800
categories: Python
tags: Python-Language
excerpt: 在容器内使用psutil
mathjax: true
typora-root-url: ../
---

# 在容器内使用psutil

启动容器的时候，加上`-v=/proc:/host/proc:ro`

在容器内部就可以这样

```python
>>> import psutil
>>> psutil.PROCFS_PATH = "/host/proc"
>>> for proc in psutil.process_iter():
...     print(proc.name().lower())
```

