---
layout: post
title:  "CPU Model的选取: host-model"
date:   2021-03-27 23:00:00 +0800
categories: Openstack
tags: Openstack-Nova
excerpt: CPU Model的选取: host-model
mathjax: true
typora-root-url: ../
---

# nova中CPU Model的选取: host-model

首先，每台hypervisor都有自己的一个CPU型号，而跑在他上面的虚拟机是怎样的CPU型号呢？这取决于nova在生成虚拟机的XML的时候是怎样定义的。定义CPU model有这些好处：

1. 可以通过把host cpu feature暴露给guest从而最大限度地提升VM的性能
2. 保证在不同机器之间cpu的一致性，移除对qemu default的依赖

在libvirt中，通过提供

1. 基本CPU Model（一组cpu feature flags的缩写）来指定CPU
2. 一组additional feature flags
3. topology (sockets/cores/threads)

libvirt KVM driver提供了一组CPU Model（定义在/usr/share/libvirt/cpu_map.xml中的）

## CPU mode

libvirt支持三种形式地去定义CPU Model，也就是三种CPU mode：

- host-passthrough: libvirt 令 KVM 把宿主机的 CPU 指令集全部透传给虚拟机。因此虚拟机能够最大限度的使用宿主机 CPU 指令集，故性能是最好的。但是在热迁移时，它要求目的节点的 CPU 和源节点的一致。
- host-model: libvirt 根据当前宿主机 CPU 指令集从配置文件 /usr/share/libvirt/cpu_map.xml 选择一种最相配的 CPU 型号。在这种 mode 下，虚拟机的指令集往往比宿主机少，性能相对 host-passthrough 要差一点，但是热迁移时，它允许目的节点 CPU 和源节点的存在一定的差异。
- custom: 这种模式下虚拟机 CPU 指令集数最少，故性能相对最差，但是它在热迁移时跨不同型号 CPU 的能力最强。此外，custom 模式下支持用户添加额外的指令集。

那在host-model下libvirt是怎么选择host匹配哪一个CPU Model呢？看一个例子

```xml
MINSU-M-M1RW:libvirt minsu$ cat src/cpu_map/x86_Skylake-Server.xml
<cpus>
  <model name='Skylake-Server'>
    <decode host='on' guest='on'/>
    <signature family='6' model='85' stepping='0-4'/> <!-- 050654 -->
    <vendor name='Intel'/>
    <feature name='3dnowprefetch'/>
    <feature name='abm'/>
    <feature name='adx'/>
    <feature name='aes'/>
    <feature name='apic'/>
    <feature name='arat'/>
    <feature name='avx'/>
    <feature name='avx2'/>
    <feature name='avx512bw'/>
    <feature name='avx512cd'/>
    <feature name='avx512dq'/>
    <feature name='avx512f'/>
    <feature name='avx512vl'/>
    <feature name='bmi1'/>
    <feature name='bmi2'/>
    <feature name='clflush'/>
    <feature name='clwb'/>
    <feature name='cmov'/>
    <feature name='cx16'/>
    <feature name='cx8'/>
    <feature name='de'/>
    <feature name='erms'/>
    <feature name='f16c'/>
    <feature name='fma'/>
    <feature name='fpu'/>
    <feature name='fsgsbase'/>
    <feature name='fxsr'/>
    <feature name='hle'/>
    <feature name='invpcid'/>
    <feature name='lahf_lm'/>
    <feature name='lm'/>
    <feature name='mca'/>
    <feature name='mce'/>
    <feature name='mmx'/>
    <feature name='movbe'/>
    <feature name='mpx'/>
    <feature name='msr'/>
    <feature name='mtrr'/>
    <feature name='nx'/>
    <feature name='pae'/>
    <feature name='pat'/>
    <feature name='pcid'/>
    <feature name='pclmuldq'/>
    <feature name='pdpe1gb'/>
    <feature name='pge'/>
    <feature name='pni'/>
    <feature name='popcnt'/>
    <feature name='pse'/>
    <feature name='pse36'/>
    <feature name='rdrand'/>
    <feature name='rdseed'/>
    <feature name='rdtscp'/>
    <feature name='rtm'/>
    <feature name='sep'/>
    <feature name='smap'/>
    <feature name='smep'/>
    <feature name='sse'/>
    <feature name='sse2'/>
    <feature name='sse4.1'/>
    <feature name='sse4.2'/>
    <feature name='ssse3'/>
    <feature name='syscall'/>
    <feature name='tsc'/>
    <feature name='tsc-deadline'/>
    <feature name='vme'/>
    <feature name='x2apic'/>
    <feature name='xgetbv1'/>
    <feature name='xsave'/>
    <feature name='xsavec'/>
    <feature name='xsaveopt'/>
  </model>
</cpus>
```

这是CPU Model x86_Skylake-Server的定义

```xml
<signature family='6' model='85' stepping='0-4'/>
```

我们再看一台主机

```shell
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                64
On-line CPU(s) list:   0-63
Thread(s) per core:    2
Core(s) per socket:    16
Socket(s):             2
NUMA node(s):          2
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 85
Model name:            Intel(R) Xeon(R) Gold 6142 CPU @ 2.60GHz
Stepping:              4
CPU MHz:               3299.987
CPU max MHz:           3700.0000
CPU min MHz:           1000.0000
BogoMIPS:              5200.00
Virtualization:        VT-x
L1d cache:             32K
L1i cache:             32K
L2 cache:              1024K
L3 cache:              22528K
NUMA node0 CPU(s):     0-15,32-47
NUMA node1 CPU(s):     16-31,48-63
Flags:                 fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc art arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc aperfmperf eagerfpu pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid dca sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch epb cat_l3 cdp_l3 invpcid_single intel_ppin intel_pt ssbd mba ibrs ibpb stibp tpr_shadow vnmi flexpriority ept vpid fsgsbase tsc_adjust bmi1 hle avx2 smep bmi2 erms invpcid rtm cqm mpx rdt_a avx512f avx512dq rdseed adx smap clflushopt clwb avx512cd avx512bw avx512vl xsaveopt xsavec xgetbv1 cqm_llc cqm_occup_llc cqm_mbm_total cqm_mbm_local dtherm ida arat pln pts hwp hwp_act_window hwp_epp hwp_pkg_req pku ospke md_clear spec_ctrl intel_stibp flush_l1d
```

CPU family:            6
Model:                 85

Stepping:              4

根据这三个就匹配到了x86_Skylake-Server

然后nova会读取virsh domcapabilities，从而生成vm的xml

有个问题是，我们在virsh domcapabilities的时候，会发现，有个feature，明明我这台主机不支持的呀，为什么会在virsh domcapabilities里呢？

这是因为

```
Features that show as enabled in domain capabilities even though they are not
available in the host capabilities are enabled and/or emulated by QEMU/KVM
regardless of what the host supports.


Libvirt tries to find a model with matching signature (family, model, stepping
numbers) if possible and uses a "shortest additional features list" heuristics
if a model with matching signature is not present in our CPU map. But we
cannot disable features present in the selected model for virsh capabilities,
which means libvirt has to show a CPU model that is fully supported by the
host (no features have to be disabled) plus a list of additional features on
top of it.
```

## Live Migration

从理论上说：

- host-passthrough: 要求源节点和目的节点的指令集完全一致
- host-model: 允许源节点和目的节点的指令集存在轻微差异
- custom: 允许源节点和目的节点指令集存在较大差异

故热迁移通用性如下：

```
custom > host-model > host-passthrough
```

从实际情况来看，公司不同时间采购的 CPU 型号可能不相同；不同业务对 CPU 型号的要求也有差异。虽然互联网多采用 intel E5 系列的 CPU，但是该系列的 CPU 也有多种型号，常见的有 Xeon，Haswell，IvyBridge，SandyBridge 等等。即使是 host-model，在这些不同型号的 CPU 之间热迁移虚拟机也可能失败。所以从热迁移的角度，在选择 host-mode 时：

- 需要充分考虑既有宿主机类型，以后采购扩容时，也需要考虑相同问题
- 除非不存在热迁移的场景，否则不应用选择 host-passthrough
- host-model 下不同型号的 CPU 最好能以 aggregate hosts 划分，在迁移时可以使用 aggregate filter 来匹配相同型号的物理机
- 如果 CPU 型号过多，且不便用 aggregate hosts 划分，建议使用 custom mode

# References

[1] [https://wiki.openstack.org/wiki/LibvirtXMLCPUModel](https://wiki.openstack.org/wiki/LibvirtXMLCPUModel)

[2] [https://bugzilla.redhat.com/show_bug.cgi?id=1839926](https://bugzilla.redhat.com/show_bug.cgi?id=1839926)

