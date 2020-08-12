---
layout: post
title:  "python操作github"
date:   2020-08-12 23:01:00 +0800
categories: Python
tags: Python-Language
excerpt: python操作github
mathjax: true
typora-root-url: ../
---

# python操作github

用了pygithub这个库

可以生成personal token来操作github

![image-20200812105505830](/../assets/images/image-20200812105505830.png)

## 生成Github对象

```python
from github import Github

base_url = 'https://wwwin-github.cisco.com/api/v3'
access_token = '**********************'
g = Github(base_url=base_url, login_or_token=access_token)
```

## 搜索repo

```python
repos = g.search_repositories("release")
>>> for x in repos:
...     print(x)
...
Repository(full_name="webex-iaas/ocp-release")
Repository(full_name="webex-iaas/ocp-releases")
Repository(full_name="qingzhhu/ocp-release-test")
```

## Get repo

```python
repo = g.get_repo('webex-iaas/ocp-releases')
```

## 下载repo中的内容

```python
repo = g.get_repo('webex-iaas/ocp-releases')
content = repo.get_contents('README.md')
url = content.download_url
r = requests.get(url)
with open('README.md', 'wb') as f:
		f.write(r.content)
```

## get branch

```python
>>> for branch in repo.get_branches():
...     print(branch)
...
Branch(name="bts")
Branch(name="master")
Branch(name="prod")
Branch(name="qa")
```

## 获取某branch下文件

```python
content = repo.get_contents('README.md', 'bts')
```

bts是branch名称

# References

[1] [https://pygithub.readthedocs.io/en/latest/github_objects/GitRelease.html](https://pygithub.readthedocs.io/en/latest/github_objects/GitRelease.html)

[2] [https://pygithub.readthedocs.io/en/latest/examples.html](https://pygithub.readthedocs.io/en/latest/examples.html)

