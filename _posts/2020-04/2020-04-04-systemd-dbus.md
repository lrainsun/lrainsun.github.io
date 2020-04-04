---
layout: post
title:  "用python dbus跟systemd对话"
date:   2020-04-04 23:00:00 +0800
categories: Python
tags: Python-Language
excerpt: 用python dbus跟systemd对话
mathjax: true
typora-root-url: ../
---

# systemd

**systemd**是[Linux](https://zh.wikipedia.org/wiki/Linux)电脑[操作系统](https://zh.wikipedia.org/wiki/作業系統)之下的一套[中央化系统及设置管理程序](https://zh.wikipedia.org/wiki/Init)（init），包括有[守护进程](https://zh.wikipedia.org/wiki/守护进程)、[程序库](https://zh.wikipedia.org/wiki/程式庫)以及应用软件。而**systemctl命令**是系统服务管理器指令。

但当我们写代码的时候，要咋办呢？我们可以使用dbus跟systemd进行对话。

# dbus

D-Bus是一种高级的进程间通信机制，它由freedesktop.org项目提供，使用GPL许可证发行。D-Bus最主要的用途是在Linux桌面环境为进程提供通信，同时能将Linux桌面环境和Linux内核事件作为消息传递到进程。D-Bus的主要概率为总线，注册后的进程可通过总线接收或传递消息，进程也可注册后等待内核事件响应，例如等待网络状态的转变或者计算机发出关机指令。

我们可以直接用`gdbus`来查看对象API

```shell
[root@rain-ocp-exporter distributed-ocp-exporter]# gdbus introspect \
> --system \
> --dest org.freedesktop.systemd1 \
> --object-path /org/freedesktop/systemd1
```

对象`/org/freedesktop/systemd1`有几个interface

```shell
node /org/freedesktop/systemd1 {
  interface org.freedesktop.DBus.Peer {
  };
  interface org.freedesktop.DBus.Introspectable {
  };
  interface org.freedesktop.DBus.Properties {
  }
  interface org.freedesktop.systemd1.Manager {
  };
```

我们主要可以用Manager interface来跟systemd service manager进行交互

```shell
  interface org.freedesktop.systemd1.Manager {
    methods:
      GetUnit(in  s arg_0,
              out o arg_1);
      GetUnitByPID(in  u arg_0,
                   out o arg_1);
      LoadUnit(in  s arg_0,
               out o arg_1);
      StartUnit(in  s arg_0,
                in  s arg_1,
                out o arg_2);
      StartUnitReplace(in  s arg_0,
                       in  s arg_1,
                       in  s arg_2,
                       out o arg_3);
      StopUnit(in  s arg_0,
               in  s arg_1,
               out o arg_2);
      ReloadUnit(in  s arg_0,
                 in  s arg_1,
                 out o arg_2);
      RestartUnit(in  s arg_0,
                  in  s arg_1,
                  out o arg_2);
      TryRestartUnit(in  s arg_0,
                     in  s arg_1,
                     out o arg_2);
      ReloadOrRestartUnit(in  s arg_0,
                          in  s arg_1,
                          out o arg_2);
      ReloadOrTryRestartUnit(in  s arg_0,
                             in  s arg_1,
                             out o arg_2);
      KillUnit(in  s arg_0,
               in  s arg_1,
               in  i arg_2);
      ResetFailedUnit(in  s arg_0);
      SetUnitProperties(in  s arg_0,
                        in  b arg_1,
                        in  a(sv) arg_2);
      StartTransientUnit(in  s arg_0,
                         in  s arg_1,
                         in  a(sv) arg_2,
                         in  a(sa(sv)) arg_3,
                         out o arg_4);
      GetJob(in  u arg_0,
             out o arg_1);
      CancelJob(in  u arg_0);
      @org.freedesktop.systemd1.Privileged("true")
      ClearJobs();
      @org.freedesktop.systemd1.Privileged("true")
      ResetFailed();
      ListUnits(out a(ssssssouso) arg_0);
      ListUnitsFiltered(in  as arg_0,
                        out a(ssssssouso) arg_1);
      ListJobs(out a(usssoo) arg_0);
      Subscribe();
      Unsubscribe();
      Dump(out s arg_0);
      DumpByFileDescriptor(out h arg_0);
      @org.freedesktop.systemd1.Privileged("true")
      CreateSnapshot(in  s arg_0,
                     in  b arg_1,
                     out o arg_2);
      @org.freedesktop.systemd1.Privileged("true")
      RemoveSnapshot(in  s arg_0);
      Reload();
      Reexecute();
      @org.freedesktop.systemd1.Privileged("true")
      Exit();
      @org.freedesktop.systemd1.Privileged("true")
      Reboot();
      @org.freedesktop.systemd1.Privileged("true")
      PowerOff();
      @org.freedesktop.systemd1.Privileged("true")
      Halt();
      @org.freedesktop.systemd1.Privileged("true")
      KExec();
      @org.freedesktop.systemd1.Privileged("true")
      SwitchRoot(in  s arg_0,
                 in  s arg_1);
      @org.freedesktop.systemd1.Privileged("true")
      SetEnvironment(in  as arg_0);
      @org.freedesktop.systemd1.Privileged("true")
      UnsetEnvironment(in  as arg_0);
      @org.freedesktop.systemd1.Privileged("true")
      UnsetAndSetEnvironment(in  as arg_0,
                             in  as arg_1);
      ListUnitFiles(out a(ss) arg_0);
      GetUnitFileState(in  s arg_0,
                       out s arg_1);
      EnableUnitFiles(in  as arg_0,
                      in  b arg_1,
                      in  b arg_2,
                      out b arg_3,
                      out a(sss) arg_4);
      DisableUnitFiles(in  as arg_0,
                       in  b arg_1,
                       out a(sss) arg_2);
      ReenableUnitFiles(in  as arg_0,
                        in  b arg_1,
                        in  b arg_2,
                        out b arg_3,
                        out a(sss) arg_4);
      LinkUnitFiles(in  as arg_0,
                    in  b arg_1,
                    in  b arg_2,
                    out a(sss) arg_3);
      PresetUnitFiles(in  as arg_0,
                      in  b arg_1,
                      in  b arg_2,
                      out b arg_3,
                      out a(sss) arg_4);
      PresetUnitFilesWithMode(in  as arg_0,
                              in  s arg_1,
                              in  b arg_2,
                              in  b arg_3,
                              out b arg_4,
                              out a(sss) arg_5);
      MaskUnitFiles(in  as arg_0,
                    in  b arg_1,
                    in  b arg_2,
                    out a(sss) arg_3);
      UnmaskUnitFiles(in  as arg_0,
                      in  b arg_1,
                      out a(sss) arg_2);
      SetDefaultTarget(in  s arg_0,
                       in  b arg_1,
                       out a(sss) arg_2);
      GetDefaultTarget(out s arg_0);
      PresetAllUnitFiles(in  s arg_0,
                         in  b arg_1,
                         in  b arg_2,
                         out a(sss) arg_3);
      AddDependencyUnitFiles(in  as arg_0,
                             in  s arg_1,
                             in  s arg_2,
                             in  b arg_3,
                             in  b arg_4,
                             out a(sss) arg_5);
      GetUnitFileLinks(in  s arg_0,
                       in  b arg_1,
                       out as arg_2);
    signals:
      UnitNew(s arg_0,
              o arg_1);
      UnitRemoved(s arg_0,
                  o arg_1);
      JobNew(u arg_0,
             o arg_1,
             s arg_2);
      JobRemoved(u arg_0,
                 o arg_1,
                 s arg_2,
                 s arg_3);
      StartupFinished(t arg_0,
                      t arg_1,
                      t arg_2,
                      t arg_3,
                      t arg_4,
                      t arg_5);
      UnitFilesChanged();
      Reloading(b arg_0);
    properties:
……
  };
```

interface中有三部分：

* methods：我们可以跟dbus对象进行交互的方法
* signals：我们怎样可以获得对应events
* properties：存储的数据

比如查看sshd service是否enabled

```shell
[root@rain-ocp-exporter distributed-ocp-exporter]# gdbus call \
>     --system \
>     --dest org.freedesktop.systemd1 \
>     --object-path /org/freedesktop/systemd1 \
>     --method org.freedesktop.systemd1.Manager.GetUnitFileState \
>     sshd.service
('enabled',)
```

# python-dbus

当使用python3的时候，可以使用`yum -y install python36-dbus`来安装python dbus库

```
import dbus
from dbus import SystemBus, SessionBus
```

Dbus有两类”channels“，一个是system serivce和system event，一个是login session

我们主要focus在SystemBus，要跟systemd通信，首先要创建一个对象

```python
>>> bus = dbus.SystemBus()
>>> systemd = bus.get_object('org.freedesktop.systemd1','/org/freedesktop/systemd1')
>>> systemd
<ProxyObject wrapping <dbus._dbus.SystemBus (system) at 0x7f2d725e1048> :1.11238 /org/freedesktop/systemd1 at 0x7f2d72640390>
```

`get_object`的第一个参数是一个systemd bus的well-known-name，第二个参数是是一个object-path

我们主要使用的是systemBus的Manager interface

```python
>>> manager = Interface(systemd, dbus_interface='org.freedesktop.systemd1.Manager')
```

manager interface中有很多方法，比如我们要查看一个服务是否被enable

```python
>>> unit_name = 'sshd.service'
>>> unit_state = manager.GetUnitFileState(unit_name)
>>> print(unit_state)
enabled
```

比如我们查看一个服务当前的状态

```python
>>> unit_name = 'sshd.service'
>>> sshd_unit = manager.GetUnit(unit_name)
>>> sshd_proxy = bus.get_object('org.freedesktop.systemd1', str(sshd_unit))
>>> sshd_proxy.Get('org.freedesktop.systemd1.Unit', 'ActiveState', dbus_interface='org.freedesktop.DBus.Properties')
dbus.String('active', variant_level=1)
```

# container内部

如果我们要在container内部跑dbus来获取容器外的service，会报这样的错误

```python
>>> bus = dbus.SystemBus()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/usr/lib64/python3.6/site-packages/dbus/_dbus.py", line 194, in __new__
    private=private)
  File "/usr/lib64/python3.6/site-packages/dbus/_dbus.py", line 100, in __new__
    bus = BusConnection.__new__(subclass, bus_type, mainloop=mainloop)
  File "/usr/lib64/python3.6/site-packages/dbus/bus.py", line 122, in __new__
    bus = cls._new_for_bus(address_or_type, mainloop=mainloop)
dbus.exceptions.DBusException: org.freedesktop.DBus.Error.FileNotFound: Failed to connect to socket /run/dbus/system_bus_socket: No such file or directory
```

起container的时候，需要把/run/dbus/system_bus_socket mount进去

```shell
docker run -itd --privileged -p 8084:8084 -v /root/distributed-ocp-exporter/config:/config -v /var/lib/nova/instances:/var/lib/nova/instances --name distributed_ocp_exporter -v=/run/dbus/system_bus_socket:/run/dbus/system_bus_socket -v=/proc:/host/proc:ro -v=/run/systemd:/run/systemd:ro registry-qa.webex.com/ocp/distributed-ocp-exporter:v1.0.0 --path.procfs=/host/proc --restart=always  --net host
```

# References

[1] [https://zignar.net/2014/09/08/getting-started-with-dbus-python-systemd/](https://zignar.net/2014/09/08/getting-started-with-dbus-python-systemd/)

[2] [https://www.freedesktop.org/wiki/Software/systemd/dbus/](https://www.freedesktop.org/wiki/Software/systemd/dbus/)

[3] [https://medium.com/@trstringer/talking-to-systemd-through-dbus-with-python-53b903ee88b1](https://medium.com/@trstringer/talking-to-systemd-through-dbus-with-python-53b903ee88b1)

