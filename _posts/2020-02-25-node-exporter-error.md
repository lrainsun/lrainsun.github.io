---
layout: post
title:  "node exporter Limit of concurrent requests reached (40), try again later"
date:   2020-02-25 23:00:00 +0800
categories: Prometheus
tags: Prometheus-Exporter
excerpt: Limit of concurrent requests reached (40), try again later
mathjax: true
typora-root-url: ../
---

# Limit of concurrent requests reached (40), try again later

今天在排查nodedownalert问题的时候，发现有几个Host，node_exporter是运行的好好的，而且host也没有异常，可以登录，尝试去curl metrics，就会提示

```shell
MINSU-M-M1RW:~ minsu$ curl ci32sjcmp021.webex.com:9100/metrics
Limit of concurrent requests reached (40), try again later.
```

```shell
[root@ci32sjcmp021 ~]# docker ps | grep node_exporter
f7375c90cbc9        registry-qa.webex.com/ocp/node-exporter:v1.2      "/bin/node_exporter …"   2 months ago        Up 12 days                              prometheus_node_exporter

[root@ci32sjcmp021 ~]# ps axf | grep node_exporter
433368 pts/2    S+     0:00                          \_ grep --color=auto node_exporter
  9845 ?        Dsl   11:27      \_ /bin/node_exporter --path.procfs /host/proc --path.sysfs /host/sys --path.rootfs /host/root --web.listen-address 10.121.29.31:9100

[root@ci32sjcmp021 ~]# ls -al /proc/9845/fd
total 0
dr-x------ 2 nfsnobody nfsnobody  0 Feb 13 02:58 .
dr-xr-xr-x 9 nfsnobody nfsnobody  0 Feb 13 02:58 ..
lrwx------ 1 nfsnobody nfsnobody 64 Feb 13 02:58 0 -> /dev/null
l-wx------ 1 nfsnobody nfsnobody 64 Feb 13 02:58 1 -> pipe:[88756]
lr-x------ 1 nfsnobody nfsnobody 64 Feb 13 16:42 10 -> /host/sys/devices/LNXSYSTM:00/device:00/ACPI000D:01/power1_average_interval
lrwx------ 1 nfsnobody nfsnobody 64 Feb 13 16:42 11 -> socket:[3282945]
lrwx------ 1 nfsnobody nfsnobody 64 Feb 13 16:42 12 -> socket:[3090362]
lr-x------ 1 nfsnobody nfsnobody 64 Feb 13 16:43 13 -> /host/sys/devices/LNXSYSTM:00/device:00/ACPI000D:01/power1_average_interval
lrwx------ 1 nfsnobody nfsnobody 64 Feb 13 16:43 14 -> socket:[3283598]
lrwx------ 1 nfsnobody nfsnobody 64 Feb 13 16:43 15 -> socket:[3090375]
lr-x------ 1 nfsnobody nfsnobody 64 Feb 13 16:43 16 -> /host/sys/devices/LNXSYSTM:00/device:00/ACPI000D:01/power1_average_interval
lr-x------ 1 nfsnobody nfsnobody 64 Feb 13 16:44 17 -> /host/sys/devices/LNXSYSTM:00/device:00/ACPI000D:01/power1_average_interval
lrwx------ 1 nfsnobody nfsnobody 64 Feb 13 16:44 18 -> socket:[3090380]
lr-x------ 1 nfsnobody nfsnobody 64 Feb 13 16:44 19 -> /host/sys/devices/LNXSYSTM:00/device:00/ACPI000D:01/power1_average_interval
l-wx------ 1 nfsnobody nfsnobody 64 Feb 13 02:58 2 -> pipe:[88757]
lrwx------ 1 nfsnobody nfsnobody 64 Feb 13 16:44 20 -> socket:[3090388]
lr-x------ 1 nfsnobody nfsnobody 64 Feb 13 16:45 21 -> /host/sys/devices/LNXSYSTM:00/device:00/ACPI000D:01/power1_average_interval
lr-x------ 1 nfsnobody nfsnobody 64 Feb 13 16:45 22 -> /host/sys/devices/LNXSYSTM:00/device:00/ACPI000D:01/power1_average_interval
```

上面可以看到Node exporter打开的fd

```shell
[root@ci32sjcmp021 ~]# ls -al /sys/class/hwmon
total 0
drwxr-xr-x  2 root root 0 Feb 25 14:22 .
drwxr-xr-x 74 root root 0 Feb 25 14:32 ..
lrwxrwxrwx  1 root root 0 Feb 25 14:22 hwmon0 -> ../../devices/pci0000:00/0000:00:01.0/0000:01:00.0/hwmon/hwmon0
lrwxrwxrwx  1 root root 0 Feb 25 14:22 hwmon1 -> ../../devices/LNXSYSTM:00/device:00/ACPI000D:00/hwmon/hwmon1
lrwxrwxrwx  1 root root 0 Feb 25 14:22 hwmon2 -> ../../devices/LNXSYSTM:00/device:00/ACPI000D:01/hwmon/hwmon2
lrwxrwxrwx  1 root root 0 Feb 25 14:22 hwmon3 -> ../../devices/platform/coretemp.0/hwmon/hwmon3
lrwxrwxrwx  1 root root 0 Feb 25 14:22 hwmon4 -> ../../devices/platform/coretemp.1/hwmon/hwmon4
[root@ci32sjcmp021 ~]# ls -al /sys/devices/LNXSYSTM:00/device:00/ACPI000D:01/
total 0
drwxr-xr-x  5 root root    0 Feb 25 14:25 .
drwxr-xr-x 31 root root    0 Feb 25 14:25 ..
lrwxrwxrwx  1 root root    0 Feb 25 14:25 driver -> ../../../../bus/acpi/drivers/power_meter
-r--r--r--  1 root root 4096 Feb 25 14:25 hid
drwxr-xr-x  3 root root    0 Feb 25 14:25 hwmon
drwxr-xr-x  2 root root    0 Feb 25 14:25 measures
-r--r--r--  1 root root 4096 Feb 25 14:25 modalias
-r--r--r--  1 root root 4096 Feb 25 14:25 name
-r--r--r--  1 root root 4096 Feb 25 14:25 path
lrwxrwxrwx  1 root root    0 Feb 25 14:25 physical_node -> ../../../platform/ACPI000D:01
drwxr-xr-x  2 root root    0 Feb 25 14:25 power
-r--r--r--  1 root root 4096 Feb 25 14:25 power1_accuracy
-rw-r--r--  1 root root 4096 Feb 25 14:25 power1_average_interval
-r--r--r--  1 root root 4096 Feb 25 14:25 power1_average_interval_max
-r--r--r--  1 root root 4096 Feb 25 14:25 power1_average_interval_min
-rw-r--r--  1 root root 4096 Feb 25 14:25 power1_average_max
-rw-r--r--  1 root root 4096 Feb 25 14:25 power1_average_min
-r--r--r--  1 root root 4096 Feb 25 14:25 power1_is_battery
-r--r--r--  1 root root 4096 Feb 25 14:25 power1_model_number
-r--r--r--  1 root root 4096 Feb 25 14:25 power1_oem_info
-r--r--r--  1 root root 4096 Feb 25 14:25 power1_serial_number
-r--r--r--  1 root root 4096 Feb 25 14:25 status
lrwxrwxrwx  1 root root    0 Feb 25 14:25 subsystem -> ../../../../bus/acpi
-rw-r--r--  1 root root 4096 Feb 25 14:25 uevent
-r--r--r--  1 root root 4096 Feb 25 14:25 uid

total 0
drwxr-xr-x  5 root root    0 Feb 25 14:25 .
drwxr-xr-x 31 root root    0 Feb 25 14:25 ..
lrwxrwxrwx  1 root root    0 Feb 25 14:25 driver -> ../../../../bus/acpi/drivers/power_meter
-r--r--r--  1 root root 4096 Feb 25 14:25 hid
drwxr-xr-x  3 root root    0 Feb 25 14:25 hwmon
drwxr-xr-x  2 root root    0 Feb 25 14:25 measures
-r--r--r--  1 root root 4096 Feb 25 14:25 modalias
-r--r--r--  1 root root 4096 Feb 25 14:25 name
-r--r--r--  1 root root 4096 Feb 25 14:25 path
lrwxrwxrwx  1 root root    0 Feb 25 14:25 physical_node -> ../../../platform/ACPI000D:01
drwxr-xr-x  2 root root    0 Feb 25 14:25 power
-r--r--r--  1 root root 4096 Feb 25 14:25 power1_accuracy
-rw-r--r--  1 root root 4096 Feb 25 14:25 power1_average_interval
-r--r--r--  1 root root 4096 Feb 25 14:25 power1_average_interval_max
-r--r--r--  1 root root 4096 Feb 25 14:25 power1_average_interval_min
-rw-r--r--  1 root root 4096 Feb 25 14:25 power1_average_max
-rw-r--r--  1 root root 4096 Feb 25 14:25 power1_average_min
-r--r--r--  1 root root 4096 Feb 25 14:25 power1_is_battery
-r--r--r--  1 root root 4096 Feb 25 14:25 power1_model_number
-r--r--r--  1 root root 4096 Feb 25 14:25 power1_oem_info
-r--r--r--  1 root root 4096 Feb 25 14:25 power1_serial_number
-r--r--r--  1 root root 4096 Feb 25 14:25 status
lrwxrwxrwx  1 root root    0 Feb 25 14:25 subsystem -> ../../../../bus/acpi
-rw-r--r--  1 root root 4096 Feb 25 14:25 uevent
-r--r--r--  1 root root 4096 Feb 25 14:25 uid
```

网上说应该跟hwmon（`hwmon`即硬件监控(`Hardware monitor`)，它是用于检测设备状态的一类传感器设备接口，比如CPU温度、风扇转速、模数转换等）有关，看了下好像果然是

这被认为是一个kernel bug，而一旦node exporter进入D（uninterruptible sleep）状态（等待I/O，不可中断，处于uninterruptible sleep状态的进程通常是在等待IO，比如磁盘IO，网络IO，其他外设IO，如果进程正在等待的IO在较长的时间内都没有响应，那么就很会不幸地被 ps看到了，同时也就意味着很有可能有IO出了问题，可能是外设本身出了故障）后，所以啥也做不了。node exporter之所以做这样一个concurrent requests的限制，就是为了避免继续建立阻塞线程

避免这个问题的方法可能是不启用hwmon

在启动node-exporter的时候，加上如下参数`--no-collector.hwmon`

# Reference

[1] [https://www.kernel.org/doc/html/latest/hwmon/acpi_power_meter.html](https://www.kernel.org/doc/html/latest/hwmon/acpi_power_meter.html)

[2] [https://github.com/prometheus/node_exporter/issues/1507](https://github.com/prometheus/node_exporter/issues/1507)

