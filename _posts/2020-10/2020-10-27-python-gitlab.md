---
layout: post
title:  "python操作gitlab"
date:   2020-10-27 23:00:00 +0800
categories: Python
tags: Python-Language
excerpt: python操作gitlab
mathjax: true
typora-root-url: ../
---

# python操作gitlab

由于产线访问不了github，我们可能会要支持gitlab，两者的api是不一样的，所以又实现了一套操作gitlab的逻辑

在gitlab里，repo叫作Project

# 获取project

```python
import gitlab
gl = gitlab.Gitlab('https://sdpgit.webex.com', private_token='***********')
projects = gl.projects.list()
for project in projects:
		print(project)
    
project = gl.projects.get('IaaS_CM/OCP/ocp_dags')
project
<Project id:230>
```

# 列出所有release、tag

```python
releases = project.releases.list()
tags = project.tags.list()
```

# 获取文件

```python
readme = project.files.get('README.md', ref='master')
content = readme.decode()
with open('readme.md', 'wb') as f:
    f.write(content)
```

# release assets

```python
assets = release.assets
for f in assets['sources']:
    if f['format'] == 'tar.gz':
        url = f['url']

url
'https://sdpgit.webex.com/iaas_cm/ocp/ocp_dags/-/archive/v0.1/ocp_dags-v0.1.tar.gz'
```

这里有个坑，蛮以为拿着这个Url就可以get到文件了，可谁知总是会redirected，认证似乎是过不了

curl --header 'PRIVATE-TOKEN: ********' "https://sdpgit.webex.com/iaas_cm/ocp/ocp_dags/-/archive/v0.1/ocp_dags-v0.1.tar.gz"
<html><body>You are being <a href="https://sdpgit.webex.com/users/sign_in">redirected</a>.</body></html>

后来才发现，这样下载文件是行不通的。参考https://docs.gitlab.com/ee/api/repositories.html#get-file-archive

curl --header "PRIVATE-TOKEN: ******" "https://sdpgit.webex.com/api/v4/projects/230/repository/archive.tar.gz"

这样就可以，而且同样的方式可以下载最新代码，release，tag，commit，其实很方便，比如：

```python
response = requests.get("https://sdpgit.webex.com/api/v4/projects/230/repository/archive.tar.gz", headers={'PRIVATE-TOKEN': '*********'}, stream=True)
with open('test.tar.gz', 'wb') as f:
    f.write(response.content)
    
    
response = requests.get("https://sdpgit.webex.com/api/v4/projects/230/repository/archive.tar.gz?sha=v0.2", headers={'PRIVATE-TOKEN': '********'}, stream=True)
with open('test.tar.gz', 'wb') as f:
    f.write(response.content)

response = requests.get("https://sdpgit.webex.com/api/v4/projects/230/repository/archive.tar.gz?sha=92cbabed9f45dd7a89d1aa818a55c46486687073", headers={'PRIVATE-TOKEN': '**********'}, stream=True)
with open('test.tar.gz', 'wb') as f:
    f.write(response.content)
```

Get an archive of the repository. This endpoint can be accessed without authentication if the repository is publicly accessible.

This endpoint has a rate limit threshold of 5 requests per minute for GitLab.com users.

```shell
GET /projects/:id/repository/archive[.format]
```

`format` is an optional suffix for the archive format. Default is `tar.gz`. Options are `tar.gz`, `tar.bz2`, `tbz`, `tbz2`, `tb2`, `bz2`, `tar`, and `zip`. For example, specifying `archive.zip` would send an archive in ZIP format.

Parameters:

- `id` (required) - The ID or [URL-encoded path of the project](https://docs.gitlab.com/ee/api/README.html#namespaced-path-encoding) owned by the authenticated user
- `sha` (optional) - The commit SHA to download. A tag, branch reference, or SHA can be used. This defaults to the tip of the default branch if not specified. For example:

```shell
curl --header "PRIVATE-TOKEN: <your_access_token>" "https://gitlab.com/api/v4/projects/<project_id>/repository/archive?sha=<commit_sha>"
```

# 创建release

```python
project.releases.create(data = {"name":"New release","tag_name":"v0.1","description":"Super nice release"}, ref='master')
<ProjectRelease tag_name:v0.1>
project.releases.list()
[<ProjectRelease tag_name:v0.1>]
```

# 获取release

```python
release = project.releases.get('v0.1')
```

