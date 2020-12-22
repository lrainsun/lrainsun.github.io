---
layout: post
title:  "git apply patch"
date:   2020-12-22 23:00:00 +0800
categories: Python
tags: Python-Language
excerpt: git apply patch
mathjax: true
typora-root-url: ../
---

# git apply patch

当我们从社区拿到一个Patch的时候，如果想要自动化地backport回来，可以这样

## git patch download

第一步先要把社区的patch download下来，download的地址就是commit的Url后面加上'.patch'

```python
    g = Github(base_url=github_connection.host, login_or_token=github_connection.password)
    repo = g.get_repo('openstack/kolla-ansible')
    commit = repo.get_commit(commit_id)
    patch_url = commit.html_url + '.patch'
    patch = requests.get(patch_url)
    with open('/tmp/' + commit_id + '.patch', 'wb') as f:
        print('***** Download patch for commit %s *****' % commit_id)
        f.write(patch.content)
```

## git clone repo

先把我们要backport的repo clone下来

```python
import git
from git.cmd import Git

git.Repo.clone_from(url=url, to_path=OCP_RELEASE)
ocp_relaese_git = Git(OCP_RELEASE)
```

## git config

然后要配置用户名和邮件，否则后面会报错

```python
    # git config
    (status, stdout, stderr) = ocp_relaese_git.execute(['git',
                                                        'config',
                                                        '--global',
                                                        'user.email',
                                                        'ocp.gen@cisco.com'],
                                                        with_extended_output=True)
    if status:
        print('***** Git config for user.email failed, error %s *****' % stderr)
        
    (status, stdout, stderr) = ocp_relaese_git.execute(['git',
                                                        'config',
                                                        '--global',
                                                        'user.name',
                                                        'ocp.gen'],
                                                        with_extended_output=True)
    if status:
        print('***** Git config for user.name failed, error %s *****' % stderr)
```

## patch apply check

然后要检查一下patch是不是可以被apply

```python
    # patch check
    (status, stdout, stderr) = ocp_relaese_git.execute(['git',
                                                        'apply',
                                                        '--check',
                                                        '--directory=kolla-ansible',
                                                        '/tmp/' + commit_id + '.patch'],
                                                       with_extended_output=True)
    if status:
        print('***** Patch check failed for commit %s, error %s *****' % (commit_id, stderr))
```

## patch apply

检查通过之后，就可以直接apply

```python
    # git apply patch
    (status, stdout, stderr) = ocp_relaese_git.execute(['git',
                                                        'am',
                                                        '--directory=kolla-ansible',
                                                        '/tmp/' + commit_id + '.patch'],
                                                       with_extended_output=True)
    if status:
        print('***** Patch apply failed for commit %s, error %s *****' % (commit_id, stderr))
```

## git push

最后一步就是把刚刚的commit push上去，Git apply的好处是可以保留社区原有patch的message跟user信息

```python
    # git push
    (status, stdout, stderr) = ocp_relaese_git.execute(['git', 'push'], with_extended_output=True)
    if status:
        print('***** Patch push failed for commit %s, error %s *****' % (commit_id, stderr))
```

![image-20201222162323365](/../assets/images/image-20201222162323365.png)