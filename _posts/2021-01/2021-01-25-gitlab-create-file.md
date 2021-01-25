---
layout: post
title:  "gitlab创建文件"
date:   2021-01-25 23:00:00 +0800
categories: Python
tags: Python-Language
excerpt: gitlab创建文件
mathjax: true
typora-root-url: ../
---

# gitlab创建文件

用python sdk操作Gitlab，我们想要commit，update文件

```python
import gitlab

gl = gitlab.Gitlab('https://sdpgit.webex.com',private_token='********')
project = gl.projects.get('IaaS_CM/OCP' + '/' + 'ocp-release')
project.files.create({'file_path':'test','content':'test','branch':'master','commit_message':'commit'})

project.files.update(file_path='test', new_data={'content': 'test', 'branch':'master','commit_message':'commit'})

project.files.get(file_path='test', ref='master')

project.commits.create({'branch':'master', 'actions':[{'action':'update','file_path':'test','content':'test'}], 'commit_message':'update'})
```

