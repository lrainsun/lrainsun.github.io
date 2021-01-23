---
layout: post
title:  "python返回类实例"
date:   2021-01-23 23:00:00 +0800
categories: Python
tags: Python-Language
excerpt: python返回类实例
mathjax: true
typora-root-url: ../
---

# python返回类实例

我们有时候需要想要封装，输入一些参数，获取一个类实例，可以用return self来实现

```python
def get_zero_touch_db_api(url, username, password, db_instance='infradb'):
    try:
        return ZTDBAPI(url, username, password, db_instance).get_db_api(), None
    except Exception as e:
        return None, e
      
class ZTDBAPI(object):
    def __init__(self, host, user, password, db_instance='infradb'):
        self.db = DBAPI(self._get_connection(host, user, password, db_instance))

    def _get_connection(self, host, user, password, db_instance):
        return '''postgresql://%s:%s@%s/%s''' % (
                user,
                password,
                host,
                db_instance,
        )
      
    def get_db_api(self):
        return self
```

