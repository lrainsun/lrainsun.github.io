---
layout: post
title:  "获取github commit"
date:   2020-12-19 23:00:00 +0800
categories: Python
tags: Python-Language
excerpt: 获取github commit
mathjax: true
typora-root-url: ../
---

# 获取github commit

我们新版本的Openstack通过Kolla-ansible来部署，会稳定在一个特定的版本，但是社区的一些bug可能是对我们也有帮助的，所以我们希望能够经常地获取到社区github上的commit，分析是不是需要backport回来

github用户登陆之后，可以生成一个accesstoken，用这个token就可以来访问github，获取到我们想要跟踪的Kolla-ansible repo

```python
from github import Github
g = Github(base_url='https://api.github.com', login_or_token=XXXXXXX)
repo = g.get_repo('openstack/kolla-ansible')
```

因为我们想要拿到的是ussuri版本的commit，所以可以这样

```python
commit_date = datetime(2020, 8, 1, 0, 0)
commits = repo.get_commits(sha='stable/ussuri', path='ansible', since=commit_date)
```

这表示获取2020.8.1之后的commit，ansible这个目录下的commit

这个commit是一个Paginated list，commits[0]是最新日期的，那我们在打印的时候，期望是从旧到新的打印，所以想要把这个List逆序一下

```python
commits.reversed
```

可是这里有个问题，逆序完了之后，用for循环来打印，却发现个数变少了，Google了一下

```
Paginated list reversed only gives the first page of results backwards #1136
```

原来逆序了之后，只显示第一页的结果了

所以这里我们先把Paginated list转成list，再做逆序就不会有这个问题了

```python
commits = list(commits)
commits.reverse()
```

```python
    for commit in commits:
        print('**************************************************')
        print('commit id: %s' % commit.sha)
        print('commit date: %s' % commit.commit.committer.date)
        print('commit url: %s' % commit.html_url)
        print('commit message: %s' % commit.commit.message)
        print('commit files: %s' % list(map(lambda c: c.filename, commit.files)))
        print('**************************************************')
```

还有一个问题，这样打印出来之后，发现有一些重复的commit，原来是个人的commit以及Merge到分支的commit，比如：

https://github.com/openstack/kolla-ansible/commit/5d09ab68417705c37e94319f713abb20017e2559

跟https://github.com/openstack/kolla-ansible/commit/5f2825943a35d80d55e3d427aa3eb32935d0cc09#diff-0cb87f727f31e5f5a59cca8a10c8f9b55622be05305d4e7e92c334f5911e1034

看起来我们需要前者，因为前者的message信息更完整

```python
def merged_commit(commit):
    if commit.commit.committer.name == 'Gerrit Code Review' or commit.commit.author.name == 'Zuul':
        return True
    return False
  
          if merged_commit(commit):
            # skip merged commit
            continue
```

