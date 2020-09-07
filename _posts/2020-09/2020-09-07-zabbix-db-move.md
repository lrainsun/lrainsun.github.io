---
layout: post
title:  "移动zabbix db"
date:   2020-09-07 23:00:00 +0800
categories: Linux
tags: Linux-Service
excerpt: 移动zabbix db
mathjax: true
typora-root-url: ../
---

# 移动zabbix db

zabbix使用的mysql db在`/var/lib/mysql`下，由于该分区disk太小，经常会满，所以想要移到`/var`下

步骤记录如下：

## 停掉zabbix服务

```shell
service zabbix-server stop
```

## 移除/var/lib/mysql分区

```shell
#UUID=1c2062a5-0224-4705-b70e-b335081e2f5b /var/lib/mysql          xfs     defaults        0 0
```

把这行从`/etc/fstab`注释掉

## umount分区

```shell
[root@ci22nycmp004 mysql]# umount /var/lib/mysql/
umount: /var/lib/mysql: target is busy.
        (In some cases useful info about processes that use
         the device is found by lsof(8) or fuser(1))
```

发现umount的时候会报错

```shell
[root@ci22nycmp004 mysql]# fuser -v /var/lib/mysql/
                     USER        PID ACCESS COMMAND
/var/lib/mysql:      root     kernel mount /var/lib/mysql
                     root      171332 ..c.. bash
```

重启host

## 移动db

```shell
mount /dev/sdb1 /tmp/mysql

tar -zcvf /var/mysql.tar.gz /tmp/mysql
```

修改目录权限

```shell
ls -al | grep mysql
drwxr-xr-x.  2 root    root       6 Jan 14  2020 mysql

chown mysql:mysql /var/lib/mysql/
```

解压移动文件

```shell
tar -zxvf /var/mysql.tar.gz -C /var/lib/mysql
mv /var/lib/mysql/tmp/mysql/* /var/lib/mysql
rm /var/lib/mysql/tmp -rf
```

## 启动mysql

```shell
service mariadb start
```

## 启动zabbix server

```shell
service zabbix-server start
```

