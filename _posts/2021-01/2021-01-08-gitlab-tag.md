---
layout: post
title:  "gitlab打tag"
date:   2021-01-08 23:00:00 +0800
categories: Python
tags: Python-Language
excerpt: gitlab打tag
mathjax: true
typora-root-url: ../
---

# gitlab打tag

## create tag & release

在gitlab界面上可以直接打tag

![image-20210108105452961](/../assets/images/image-20210108105452961.png)

填好tag name, create from之后，tag就能创建出来，但是发现在release的地方没有create release的地方

后来发现啊，在创建tag的时候，要填release notes，如果release notes的地方填了内容，那么就会同步创建一个同名的release，不然就不会。然后删除release是不会删除tag的，而删除tag是会同步删除release的

我们通过代码可以这样创建

```python
import gitlab
gl =  gitlab.Gitlab('https://sdpgit.webex.com',private_token='********')
project = gl.projects.get('IaaS_CM/OCP' + '/' + 'ocp-release')
tag = project.tags.create({'tag_name': 'test', 'ref': 'master'})  # create tag
tag.set_release_description('test')   # create release
```

## release/tag branch

还有一个事就是，release跟tag其实是跟branch无关的

```
Tags point to commits, and branches point to commits. A single commit can be pointed at (or be a parent of) dozens of different branches; there is no way to narrow down one specific branch as "the owner of this tag". The branch might have been deleted from upstream before you fetched it, and only the commit remains, as another example of why this can't work.
```

在我们的场景里，想要取到不同branch的最新release，先我们可以暂时这样做是因为这两个branch的内容其实是完全不相干的，所以不会有相同的commit同时提交到两个branch

```python
            project = projects.get(repo_name)
            releases= project.releases.list()
            for release in releases:
                commit_id = release.attributes['commit']['id']
                commit = project.commits.get(commit_id)
                if commit.refs(type='branch')[0]['name'] == branch:
                    return release.name
```

