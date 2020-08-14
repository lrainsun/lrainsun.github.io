---
layout: post
title:  "获取github release"
date:   2020-08-14 23:00:00 +0800
categories: Python
tags: Python-Language
excerpt: 获取github release
mathjax: true
typora-root-url: ../
---

# 获取github release

![image-20200814155539591](/../assets/images/image-20200814155539591.png)

github release通常是通过release的形式，如https://github.com/openstack/kolla-ansible/releases

标签（tag）是特定提交（commit)一个指针，也就是每个tag对应一个特定的commit。

Release是具有changelogs和二进制文件的一级对象，它可以代表超出Git架构本身的一个特定时间点之前的所有项目历史。也就是通过release，不但能够通过源码体现出项目历史，还能通过已经编译好的二进制文件来进一步描述此时的项目状态。

标签是git中的概念，而release则是Github、码云等源码托管商所提供的更高层的概念。也就是说git本身是没有release这个概念的，只有tag。

两者之间的关系则是，release基于tag，为tag添加更丰富的信息，一般是编译好的文件。

当我们发布一个release的时候，会把source code打包成zip & tar.gz的格式

![image-20200814160016717](/../assets/images/image-20200814160016717.png)

而我们想要取得某一个release具体的内容的时候，其实就可以下载这两个文件

通过release的tarball_url获取到下载的url

```python
>>> repo = g.get_repo('webex-iaas/ocp-release')
>>> release = repo.get_release('v4.0.0-beta0')
>>> release
GitRelease(title="v4.0.0-beta0 release")
>>> release.tarball_url
'https://wwwin-github.cisco.com/api/v3/repos/webex-iaas/ocp-release/tarball/v4.0.0-beta0'
>>> tarball_url = release.tarball_url
>>> response = requests.get(tarball_url, auth=('***********', 'x-oauth-basic'), stream=True)
>>> with open('ansible.tar.gz', 'wb') as f:
...     f.write(response.content)
...
807297
```

然后解压获得源码

```python
>>> import tarfile
>>> import os
>>> t = tarfile.open('ansible.tar.gz')
>>> t.extractlall(path = '/Users/minsu/Documents/minsu/ansible')
```

解压后会放到一个文件夹，我们可以把这个文件夹重命名成我们想要的

```python
>>> dir_name = t.getnames()[0]
'webex-iaas-ocp-release-31460224eaddc15b935edca92b6a94a36fac60c3'

>>> os.rename(dir_name, 'ansible')
```

