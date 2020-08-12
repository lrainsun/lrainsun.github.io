---
layout: post
title:  "logrotate"
date:   2020-08-07 23:00:00 +0800
categories: Linux
tags: Linux-Service
excerpt: logrotate
mathjax: true
typora-root-url: ../
---

# logrotate

logrotate程序是一个日志文件管理工具。用于分割日志文件，删除旧的日志文件，并创建新的日志文件，起到“转储”作用。可以节省磁盘空间。而logrotate是系统自动完成的，脚本在/etc/cron.daily/logrotate

```shell
(cron)[root@bmaas /]$ cat /etc/cron.daily/logrotate
#!/bin/sh

/usr/sbin/logrotate -s /var/lib/logrotate/logrotate.status /etc/logrotate.conf
EXITVALUE=$?
if [ $EXITVALUE != 0 ]; then
    /usr/bin/logger -t logrotate "ALERT exited abnormally with [$EXITVALUE]"
fi
exit 0
```

除此之外，weekly, hourly也可以放定时任务

```shell
(cron)[root@bmaas /]$ cat /etc/cron.
cron.d/       cron.daily/   cron.deny     cron.hourly/  cron.monthly/ cron.weekly/
```

status里可以查到logrotate的状态

```shell
(cron)[root@bmaas /]$ cat /var/lib/logrotate/logrotate.status
logrotate state -- version 2
"/var/log/kolla/ironic/ironic-dbsync.log" 2020-4-2-15:0:0
"/var/log/yum.log" 2020-4-2-15:0:0
"/var/log/kolla/ironic/ironic-ipxe-access.log" 2020-4-2-15:0:0
"/var/log/kolla/iscsi/iscsi.log" 2020-4-2-15:0:0
"/var/log/kolla/ironic/ironic-ipxe-error.log" 2020-4-2-15:0:0
"/var/log/kolla/ironic/ironic-api.log" 2020-8-3-3:11:1
"/var/log/kolla/ironic-inspector/ironic-inspector.log" 2020-8-2-3:19:1
"/var/log/kolla/ansible.log" 2020-4-2-15:0:0
"/var/log/kolla/ironic/ironic-conductor.log" 2020-8-5-14:30:1
"/var/log/iscsiuio.log" 2020-4-2-15:0:0
"/var/log/kolla/rabbitmq/rabbit@bmaas_upgrade.log" 2020-4-2-15:0:0
"/var/log/kolla/haproxy/haproxy.log" 2020-4-2-15:0:0
"/var/log/kolla/rabbitmq/rabbit@bmaas.log" 2020-4-2-15:0:0
"/var/log/kolla/chrony/tracking.log" 2020-4-2-15:0:0
"/var/log/kolla/mariadb/mariadb.log" 2020-4-2-15:0:0
"/var/log/kolla/keepalived/keepalived.log" 2020-4-2-15:0:0
```

也可以手工强制rotate，-d可以先验证一下，但不真的工作

```shell
(cron)[root@bmaas /]$ /usr/sbin/logrotate -s /var/lib/logrotate/logrotate.status -d -f /etc/logrotate.conf
reading config file /etc/logrotate.conf
including /etc/logrotate.d
reading config file ansible.conf
reading config file chrony.conf
reading config file haproxy.conf
reading config file ironic.conf
reading config file ironic-inspector.conf
reading config file iscsid.conf
reading config file iscsiuiolog
reading config file keepalived.conf
reading config file mariadb.conf
reading config file rabbitmq.conf
reading config file yum
Allocating hash table for state file, size 15360 B

Handling 11 logs

rotating pattern: "/var/log/kolla/ansible.log"
 forced from command line (6 rotations)
empty log files are not rotated, only log files >= 31457280 bytes are rotated, log files >= 104857600 are rotated earlier, old logs are removed
switching euid to 0 and egid to 42400
considering log /var/log/kolla/ansible.log
  log needs rotating
rotating log /var/log/kolla/ansible.log, log->rotateCount is 6
dateext suffix '-20200806'
glob pattern '-[0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]'
previous log /var/log/kolla/ansible.log.1 does not exist
renaming /var/log/kolla/ansible.log.6.gz to /var/log/kolla/ansible.log.7.gz (rotatecount 6, logstart 1, i 6),
renaming /var/log/kolla/ansible.log.5.gz to /var/log/kolla/ansible.log.6.gz (rotatecount 6, logstart 1, i 5),
renaming /var/log/kolla/ansible.log.4.gz to /var/log/kolla/ansible.log.5.gz (rotatecount 6, logstart 1, i 4),
renaming /var/log/kolla/ansible.log.3.gz to /var/log/kolla/ansible.log.4.gz (rotatecount 6, logstart 1, i 3),
renaming /var/log/kolla/ansible.log.2.gz to /var/log/kolla/ansible.log.3.gz (rotatecount 6, logstart 1, i 2),
renaming /var/log/kolla/ansible.log.1.gz to /var/log/kolla/ansible.log.2.gz (rotatecount 6, logstart 1, i 1),
renaming /var/log/kolla/ansible.log.0.gz to /var/log/kolla/ansible.log.1.gz (rotatecount 6, logstart 1, i 0),
copying /var/log/kolla/ansible.log to /var/log/kolla/ansible.log.1
truncating /var/log/kolla/ansible.log
removing old log /var/log/kolla/ansible.log.7.gz
error: error opening /var/log/kolla/ansible.log.7.gz: No such file or directory
switching euid to 0 and egid to 0
```

手工需要-f强制切割

```shell
(cron)[root@bmaas /]$ cat /var/l/usr/sbin/logrotate -s /var/lib/logrotate/logrotate.status -f /etc/logrotate.conf
(cron)[root@bmaas /]$ ls -alh /var/log/kolla/ironic
total 159M
drwxr-sr-x. 2 ironic   ironic 4.0K Aug  6 03:08 .
drwxrwsr-x. 9 td-agent kolla   157 Aug  6 03:08 ..
-rw-r--r--. 1 ironic   ironic 1.4M Aug  6 03:08 ironic-api.log
-rw-r--r--. 1 ironic   ironic 120M Aug  6 03:08 ironic-api.log.1
-rw-r--r--. 1 ironic   ironic 165K Aug  6 03:08 ironic-conductor.log
-rw-r--r--. 1 ironic   ironic  38M Aug  6 03:08 ironic-conductor.log.1
-rw-r--r--. 1 ironic   ironic    0 Aug  6 03:08 ironic-dbsync.log
-rw-r--r--. 1 ironic   ironic  690 Aug  6 03:08 ironic-dbsync.log.1
-rw-r--r--. 1 ironic   ironic    0 Aug  6 03:08 ironic-ipxe-access.log
-rw-r--r--. 1 ironic   ironic  293 Aug  6 03:08 ironic-ipxe-access.log.1
-rw-r--r--. 1 ironic   ironic    0 Aug  6 03:08 ironic-ipxe-error.log
-rw-r--r--. 1 ironic   ironic  247 Aug  6 03:08 ironic-ipxe-error.log.1
```

/etc/logrotate.conf下是默认值

```shell
(cron)[root@bmaas /]$ cat /etc/logrotate.conf
weekly

rotate 6

copytruncate

compress

delaycompress

notifempty

missingok

minsize 30M

maxsize 100M

su root kolla

include /etc/logrotate.d
```

/etc/logrotate.d/ 目录中的所有文件都会加载进来

 