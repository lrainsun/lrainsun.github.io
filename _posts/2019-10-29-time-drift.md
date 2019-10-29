---
layout: post
title:  "Time Drift after Live Migration"
date:   2019-10-29
categories: Openstack
tags: Openstack-Nova
excerpt: Research on time drift after live migration in openstack
mathjax: true
---

# Know About Time

First, let's discuss about the question:   "What is time?" & "What is standard time?"

Time is important as "Time is money" 🙂. If someone is willing to spend time with you, that should be a true friend. Time is also important for computer, internet...

We always use ***GMT*** or ***UTC*** as a time standard. Although GMT and UTC share the same current time in practice, there is a basic difference between the two:

* [**GMT is a time zone**](https://www.timeanddate.com/time/zones/gmt) officially used in some European and African countries. The time can be displayed using both the 24-hour format (0 - 24) or the 12-hour format (1 - 12 am/pm).
* [**UTC is not a time zone**](https://www.timeanddate.com/time/aboututc.html), but a time standard that is the basis for civil time and time zones worldwide. This means that no country or territory officially uses UTC as a local time.

In Linux, there are two kinds of clocks:

* System(Software) Clock: 
  * Counts the number of seconds past Jan 1, 1970
  * It does not exist when the system is not running
* Hardware Clock: 
  * It's the clock on host hardware
  * The hardware clock can be set from the BIOS setup screen or from whatever operating system is running
  * It keeps track of time when the system is turned off but is not used when the system is running. 

As time is so important, so how to synchonize time? NTP is the most often used protocol.

# NTP (Network Time Protocol)

NTP is a protocol designed to synchronize the clocks of computers over a network. It implementations send and receive timestamps using the UDP on port number 123.

NTP uses a hierarchical, semi-layered system of time sources. Each level of this hierarchy is termed a *stratum* and is assigned a number starting with zero for the reference clock at the top. A server synchronized to a stratum *n* server runs at stratum *n* + 1. The number represents the distance from the reference clock and is used to prevent cyclical dependencies in the hierarchy. The upper limit for stratum is 15; stratum 16 is used to indicate that a device is unsynchronized.

## Here are some configurations for ntp

* /etc/ntp.conf

  * NTP daemon configuration file

  * ```
    [root@rain-ntp-test ~]# cat /etc/ntp.conf
    #NTP Client
    server 10.225.28.1
    restrict default noserve noquery nomodify
    restrict 127.0.0.1
    restrict ::1
    restrict 10.225.28.1 mask 255.255.255.255 nomodify notrap noquery
    driftfile /var/lib/ntp/drift
    ```

* /etc/sysconfig/ntpd

  * ntpd service environment file

  * ```
    [root@rain-ntp-test ~]# cat /etc/sysconfig/ntpd
    # Command line options for ntpd
    OPTIONS="-g -x"
    ```

* /etc/localtime

  * configuration file for localtime

  * ```
    [root@rain-ntp-test ~]# cat /etc/localtime
    TZif2GMTTZif2GMT
    GMT0
    ```

## Some useful commands

* /bin/date

  * set & list linux system clock

  * ```
    [root@rain-ntp-test ~]# date
    Tue Oct 29 05:21:52 GMT 2019
    
    [root@rain-ntp-test ~]# date -u
    Tue Oct 29 05:21:54 UTC 2019
    
    [root@rain-ntp-test ~]# date +%s
    1572326517
    ```

* /sbin/hwclock

  * set & list linux hardware clock

  * ```
    [root@rain-ntp-test ~]# hwclock
    Tue 29 Oct 2019 05:22:21 AM GMT  -0.347645 seconds
    
    [root@rain-ntp-test ~]# hwclock --utc
    Tue 29 Oct 2019 05:22:26 AM GMT  -0.325390 seconds
    
    [root@rain-ntp-test ~]# hwclock --localtime
    Tue 29 Oct 2019 05:22:30 AM GMT  -0.842418 seconds
    ```

* /bin/timedatectl

  * set & list timezone  

  * ```
    [root@rain-ntp-test ~]# timedatectl
          Local time: Tue 2019-10-29 05:22:40 GMT
      Universal time: Tue 2019-10-29 05:22:40 UTC
            RTC time: Tue 2019-10-29 05:22:40
           Time zone: GMT (GMT, +0000)
         NTP enabled: yes
    NTP synchronized: no
     RTC in local TZ: yes
          DST active: n/a
    ```

* /bin/tzselect

  * time zone configuration

* /usr/sbin/ntpd

  * network time service, use /etc/ntp.conf as configuration file

* /usr/sbin/ntpdate

  * sets the local date and time by polling the Network Time Protocol (NTP) server(s) given as the server arguments to determine the correct time

## Check if ntp works well on the compute

* ntpstat

  * ```
    [root@rain-ntp-test ~]# ntpstat
    synchronised to NTP server (10.225.28.1) at stratum 10
       time correct to within 22 ms
       polling server every 512 s
    ```

* ntpq -p

  * ```
    [root@rain-ntp-test ~]# ntpq -p
         remote           refid      st t when poll reach   delay   offset  jitter
    ==============================================================================
    *10.225.28.1     10.224.236.6     9 u  101  512  377    0.598   -2.223   0.249 
    ```


# Why time drift after live migation

When migrating VMs, the VM will be paused for a while but not restarted, so there can be a time difference of several seconds or more when the VM resumes. The problem also exists when we pause a VM. Linux guest which is paused via the hypervisor, does not longer "catch up" with realtime. The guests time stays behind for the duration of the pause.

## Hypervisor/VM 

It is recommended to enable NTP on the VM guest in addition to the hypervisor host because it is not guaranteed that NTP will keep the hwclock and the system clock in sync *on the host*.

Libvirt is unable to make such decision whether guest time should be synced when doing domain resume. Libvirt knows nothing about guest time policy, whether it's using NTP or other mechanisms to sync time. 

A Request for Feature Enhancement (RFE) has been filed to add "time sync" functionality in a future release of `qemu-guest-agent` available for RHV virtual machines. 

## NTP Slew Mode

```
[root@rain-ntp-test ~]# cat /etc/sysconfig/ntpd
# Command line options for ntpd
OPTIONS="-g -x"
```

"-x" means that we are now using slew mode for ntpd. There are two modes:

* step mode
* slew mode

Step mode is used by default.  The time is slewed if the offset is less than the step threshold, is which *128ms* in step mode by default, and stepped if above the threshold. 

The "-x" option set the mode to slew mode, the step threshold is set to *600s*. 

So with -x, if the time is off by up to 600s, ntpd will gradually adjust it, rather than step it. If the offset exceeds 600s, ntpd will step it. Without -x, ntpd will step the time if the offset is greater than 128ms. In both cases (with and without -x), ***ntpd will only step the time after the step threshold has been exceeded*** (default 900s). To force ntp to step sooner, you can set the stepout option.

And since the slew rate of typical Unix kernels is limited to 0.5 ms/s, each second of adjustment requires an amortization interval of 2000 s. Thus, an adjustment as much as 600s will take almost 14 days to complete.

For "-g" option: Normally,  ntpd exits with a message to the system log if the offset exceeds the panic threshold, which is 1000 s by default. This option allows the time to be set to any value without restriction; however, this can happen only once. If the threshold is exceeded after that,  ntpd will exit with a message to the system log.

Other configuration is panic threshold: Specifies the panic threshold in seconds with default 1000s. If set to zero,the panic sanity check is disabled and a clock offset of any value will be accepted.

So, here we come to a conclusion:

1. with "-x" option, ntpd will gradually adjust the time if the offset is less than 600s
2. with "-x" option, ntpd will step the time if the offset is greater than 600s
3. ntpd will step the time after 900s (default value) as stepout threshold not configured
4. ntpd service will exit if offset (time between local & ntp server) exceed 1000s twice (default value) as panic threshold not configured

Something can be done to improve the situation (inside the VM):

1. remove "-x" option from /etc/sysconfig/ntpd
2. set "tinker setpout" options in /etc/ntp.conf: tinker stepout 10
3. set "tinker panic" options in /etc/ntp.conf: tinker panic 0   

## Use More NTP Servers

Four NTP servers are recommended. See http://support.ntp.org/bin/view/Support/SelectingOffsiteNTPServers

*"With two, it is impossible to tell which one is better, because you don't have any other references to compare them with. This is actually the worst possible configuration -- you'd be better off using just one upstream time server and letting the clocks run free if that upstream were to die or become unreachable."*

For NTP servers more than one, ntpd has built-in algorithms to select the best clock source. By using the ‘true’ option, these algorithms are bypassed. 

*"If two NTP servers are required for redundancy, one server can be specified as the "primary" NTP server by using the true option. This will force that server to always be selected and for NTP client to follow it blindly.* 

We only use one NTP server in our case, it's better configure more. 

# References

[1] [http://tldp.org/HOWTO/Clock-2.html](http://tldp.org/HOWTO/Clock-2.html)

[2] [https://en.wikipedia.org/wiki/Network_Time_Protocol](https://en.wikipedia.org/wiki/Network_Time_Protocol)

[3] [https://www.redhat.com/en/blog/avoiding-clock-drift-vms](https://www.redhat.com/en/blog/avoiding-clock-drift-vms)

[4] [https://access.redhat.com/solutions/27865](https://access.redhat.com/solutions/27865)