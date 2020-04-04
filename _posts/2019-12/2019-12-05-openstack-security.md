---
layout: post
title:  "openstack security group规则不生效"
date:   2019-12-05 23:59:00 +0800
categories: Openstack
tags: Openstack-neutron
excerpt: openstack security group规则不生效
mathjax: true
---

问题是从发现设置的security group规则不生效产生的：

* OpenStack sg设置也都是对的
* 查看iptables，chain&rule都是对的

后来发现是sysctl的bridge-nf-call-iptables设置为0（disable），导致基于bridge的iptables规则无法生效，安全组规则自然也就无效了

```shell
[root@ci92hf1cmp015 ~]# sysctl -n net.bridge.bridge-nf-call-iptables
0
```

但始终没找到是谁设置了这个值

有一些发现是：

* sysctl默认会把这个设置为0

  ```shell
  [root@ci92hf1cmp015 ~]# cat /usr/lib/sysctl.d/00-system.conf
  # Kernel sysctl configuration file
  #
  # For binary values, 0 is disabled, 1 is enabled.  See sysctl(8) and
  # sysctl.conf(5) for more details.
  
  # Disable netfilter on bridges.
  net.bridge.bridge-nf-call-ip6tables = 0
  net.bridge.bridge-nf-call-iptables = 0
  net.bridge.bridge-nf-call-arptables = 0
  ```

* `service systemd-sysctl restart`会读取这个默认值，设置为0，但我们没有发现重启该服务的情况

* neutron-openvswitch-agent启动，在prestart的时候会load br_netfilter这个module，并置成1。看起来启动顺序，neutron-openvswitch-agent在systemd-sysctl之后

  ```shell
  [root@ci92hf1cmp015 ~]# cat /usr/bin/neutron-enable-bridge-firewall.sh
  #!/bin/sh
  
  # This script is triggered on every ovs/linuxbridge agent start. Its intent is
  # to make sure the firewall for bridged traffic is enabled before we start an
  # agent that may atttempt to set firewall rules on a bridge (a common thing for
  # linuxbridge and ovs/hybrid backend setup).
  
  # before enabling the firewall, load the relevant module
  /usr/sbin/modprobe bridge
  
  # on newer kernels (3.18+), sysctl knobs are split into a separate module;
  # attempt to load it, but don't fail if it's missing (f.e. when running against
  # an older kernel version)
  /usr/sbin/modprobe br_netfilter 2>> /dev/null || :
  
  # now enable the firewall in case it's disabled (f.e. rhel 7.2 and earlier)
  for proto in arp ip ip6; do
      /usr/sbin/sysctl -w net.bridge.bridge-nf-call-${proto}tables=1
  done
  ```

* 最初还怀疑是enable了telemetry，区别在于telemetry会安装docker，安装docker后也同样会设置该值为1

后来发现这个值最终会生效在👇，可以查看被修改的时间

```shell
[root@ci92hf1cmp015 ~]# cat /proc/sys/net/bridge/bridge-nf-call-iptables
0

[root@ci92hf1cmp015 ~]# ls -al --time-style=full /proc/sys/net/bridge/ | grep bridge-nf-call-iptables
-rw-r--r-- 1 root root 0 2019-12-05 07:29:34.464347528 +0000 bridge-nf-call-iptables
```

然后发现该值被修改的时间正是我们做变更跑ansible的时间点，再检查系统日志，在该时间点前后查看，最后锁定

```shell
/sbin/tuned-adm profile ocp-tuned
```

这句执行完之后，值马上会被改成0

```shell
[root@ci92hf1cmp015 ~]# cat /etc/tuned/ocp-tuned/tuned.conf
[main]
summary=OCP tune profile, child of throughput-performance include extra config of force_latency=1
include=throughput-performance

[cpu]
force_latency=1
```

ocp-tuned里会把profile设置成`throughput-performance`，并`force_latency=1`

除去`force_latency=1`，该问题依然会复现，也就是`/sbin/tuned-adm profile throughput-performance`依然会导致问题的发生

那这又是为什么呢？后来发现tuned-adm应该是会去读sysctl的配置，因为我们没有配置过该值，所以我们最初发现的`/usr/lib/sysctl.d/00-system.conf`里有一个默认值，那么就会被设置成0了

那么解决方法，可以增加一个配置文件比如`70-ocp.conf`，在那个里面把`bridge-nf-call-iptables`设置为1，这样不管哪个service来load这个配置都会读到1了