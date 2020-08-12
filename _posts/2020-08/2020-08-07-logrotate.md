---
layout: post
title:  "linux rm文件名带空格的文件"
date:   2020-08-11 23:00:00 +0800
categories: Linux
tags: Linux-Service
excerpt: linux rm文件名带空格的文件
mathjax: true
typora-root-url: ../
---

# linux rm文件名带空格的文件

```shell
[root@bmaas mysql_backup]# rm ironic_2020-08-07 07:02:32.sql
rm: cannot remove ‘ironic_2020-08-07’: No such file or directory
rm: cannot remove ‘07:02:32.sql’: No such file or directory
```

如果一个文件名带了空格，在删除的时候会被认为是两个文件

需要这样删除

```shell
[root@bmaas mysql_backup]# rm "ironic_2020-08-07 07:02:32.sql"
rm: remove regular file ‘ironic_2020-08-07 07:02:32.sql’? y
```

