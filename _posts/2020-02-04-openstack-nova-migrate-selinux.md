---
layout: post
title:  "由于selinux没有disable导致vm live migate失败"
date:   2020-02-04 17:00:00 +0800
categories: Openstack
tags: Openstack-Nova
excerpt: 由于selinux没有disable导致vm live migate失败
mathjax: true
---

# live migrate失败

live migrate失败，查看nova compute日志可以看到：

```shell
2020-02-03 17:10:02.015 75917 INFO nova.virt.libvirt.driver [req-ce87db1d-722d-4060-b4ce-a306f4b9174b - - - - -] Connection event '0' reason 'Connection to libvirt lost: 0'
2020-02-03 17:10:19.006 75917 INFO nova.virt.libvirt.driver [req-200b08c9-e065-4026-bd17-f21792e79b53 10b45def459f4bd1995b776542fde3e6 f974270c43f741d8abfb347199395943 - - -] Connection event '1' reason 'None'
2020-02-03 17:10:43.981 75917 ERROR nova.virt.libvirt.driver [req-200b08c9-e065-4026-bd17-f21792e79b53 10b45def459f4bd1995b776542fde3e6 f974270c43f741d8abfb347199395943 - - -] [instance: 223cfd1d-b3b4-4eaa-8b44-a85d97893d90] Live Migration failure: unsupported configuration: Unable to find security driver for model selinux
```

按理说selinux应该是被disable的，但是

```shell
[root@ci22nycmp007 ~]# getenforce
Permissive
```

在selinux是permissive状态的时候，在host上的vm的seclabel mode是selinux，在往其他host迁移的时候，如果对端的host没有启用selinux就会live migarate失败

```shell
[root@ci22nycmp007 ~]# virsh dumpxml instance-00000ade | grep seclabel
  <seclabel type='dynamic' model='selinux' relabel='yes'>
  </seclabel>
  <seclabel type='dynamic' model='dac' relabel='yes'>
  </seclabel>
```

查看selinux配置

```shell
[root@ci22nycmp007 ~]# cat /etc/selinux/config

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of three values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

是disabled状态，但我们实际是permissive状态，原因是，配置修改后，需要reboot host，才能够变成disable状态

当我们通过/usr/sbin/setenforce 0 =》permissive =》 change config => reboot host => disable

目前的情况，已经有vm在上面，可以

1. cold migrate vm到其他host，然后 change config, reboot本host
2. stop vm => change config => reboot host => start vm