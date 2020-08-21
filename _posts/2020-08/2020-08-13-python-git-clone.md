---
layout: post
title:  "python github clone"
date:   2020-08-13 23:00:00 +0800
categories: Python
tags: Python-Language
excerpt: python github clone
mathjax: true
typora-root-url: ../
---

# python github clone

发现pygithub库并没有找到git clone的方法，但是可以获取到git的clone url

```python
repo = g.get_repo('webex-iaas/ocp-releases')
repo.clone_url
'https://wwwin-github.cisco.com/webex-iaas/ocp-releases.git'
```

后来找到，使用gitpython是可以的，`pip install gitpython`可以安装

两种方式都可以

```python
import git
git.Git('/Users/minsu/Documents/minsu/github').clone('https://access_token:x-oauth-basic@wwwin-github.cisco.com/webex-iaas/ocp-releases.git')
git.Git('/Users/minsu/Documents/minsu/github').clone('https://access_token:x-oauth-basic@wwwin-github.cisco.com/webex-iaas/ocp-releases', branch='master')

g = git.cmd.Git('/Users/minsu/Documents/minsu/github/ocp-releases')
g.pull()
```

or

```python
from git import Repo
repo = Repo.clone_from('https://access_token:x-oauth-basic@wwwin-github.cisco.com/webex-iaas/releases.git', '/Users/minsu/Documents/minsu/github', branch='master')
```

checkout某个commit

```python
repo = git.Repo.clone_from('https://access_token:x-oauth-basic@wwwin-github.cisco.com/webex-iaas/ocp-release.git', '/Users/minsu/Documents/minsu/github/ocp-release')
repo.git.checkout('55798aa6560e6586afbed76fd5a804165bac0412')
```

