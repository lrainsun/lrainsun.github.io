---
layout: post
title:  "从远程主机上下载文件夹"
date:   2020-06-12 23:00:00 +0800
categories: Airflow
tags: Airflow-Tutorial
excerpt: 从远程主机上下载文件夹
mathjax: true
typora-root-url: ../
---

# 从远程主机上下载文件夹

我们要下载inventory的时候，还有一个host_vars需要下载，这么多文件一个一个下载就不合适了，所以需要一个从远程主机下载文件的方法

万能的google告诉我们，用sftp可以实现的

```python
import paramiko
import stat

def __get_all_files_in_remote_dir(sftp, remote_dir):
    all_files = list()

    if remote_dir[-1] == '/':
        remote_dir = remote_dir[0:-1]

    files = sftp.listdir_attr(remote_dir)
    for x in files:
        filename = remote_dir + '/' + x.filename
        if stat.S_ISDIR(x.st_mode):
            all_files.extend(__get_all_files_in_remote_dir(sftp, filename))
        else:
            all_files.append(filename)
    return all_files

def sftp_get_dir(ip, remote_dir, local_dir):
    t = paramiko.Transport(sock=(ip, 22))
    t.connect(username='ocpadmin', password='*******')
    sftp = paramiko.SFTPClient.from_transport(t)

    all_files = __get_all_files_in_remote_dir(sftp, remote_dir)

    for x in all_files:
        filename = x.split('/')[-1]
        local_filename = os.path.join(local_dir, filename)
        sftp.get(x, local_filename)
        print('Download %s from %s to %s success!' % (remote_dir, ip, local_filename))
```

