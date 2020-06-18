---
layout: post
title:  "计算kvm vm的steal time"
date:   2020-04-17 23:00:00 +0800
categories: Linux
tags: Linux-Service
excerpt: 计算kvm vm的steal time
mathjax: true
typora-root-url: ../
---

# steal time

之前有学习过[cpu contention](https://lrainsun.github.io/2020/04/01/cpu-contention/)，有讲过可以通过steal time(stolen time)来反应cpu contention的情况。所以我们想要做个metrics，获取所有vm的这个值，后期可以做一些分析。有了这个值之后我们就可以判断一台hypervisor上运行这么多vm是不是合理，可以调整配置。

但是进到vm里top获取到vm内部的steal time不太现实，有个开源工具[kvmtop](https://github.com/cha87de/kvmtop)通过计算获取到了这个值，但这个工具我们要用起来，还得每个hypervisor都安装，并跑起来，再通过exporter去读取数据，有点麻烦。kvmtop是用go来实现的，我们可以把逻辑看明白了，转换到python。因为我们已经有一个自定义的distributed exporter分在所有hypervisor上了。如果代码看明白了，可以直接加到这个exporter里就好了。

需要mount `-v /var/run/libvirt/libvirt-sock:/var/run/libvirt/libvirt-sock`

# 步骤

## 发现对象 —— kvm process

第一步，通过libvirt来list所有domain，并且把domain的name啊，uuid啥的都记录下来。再去找到每个domain对应pid，把对象记录下来，kvmtop是这样做的：

```go
// query libvirt
doms, err := connector.Libvirt.Connection.ListAllDomains(libvirt.CONNECT_LIST_DOMAINS_ACTIVE)
if err != nil {
   log.Printf("Cannot get list of domains form libvirt.")
   return
}

// update process list
processes = util.GetProcessList()
```

```go
// GetProcessList reads and returns all PIDs from the proc filesystem
func GetProcessList() []int {
   files, err := ioutil.ReadDir(config.Options.ProcFS)
   if err != nil {
      log.Fatal(err)
   }

   var processes []int
   for _, f := range files {
      // is it a folder?
      if !f.IsDir() {
         continue
      }
      // is the name a number?
      if pid, err := strconv.Atoi(f.Name()); err == nil {
         processes = append(processes, pid)
      }
   }

   return processes
}
```

这里把系统所有的/proc/下，只要是数字的目录，process都list出来了

```go
// lookup PID
var pid int
for _, process := range processes {
   cmdline := util.GetCmdLine(process)
   if cmdline != "" && strings.Contains(cmdline, name) {
      // fmt.Printf("Found PID %d for instance %s (cmdline: %s)", process, name, cmdline)
      pid = process
      break
   }
}
domain.PID = pid
```

然后又更新了一把domain，根据每个process的`/proc/pid/cmdline` cmdline，去找里面是不是包含了domain name，比如下面是一个vm Process的cmdline，如果包含那说明就是这个pid，把pid更新进去。

```shell
[root@ci81hf1cmp007 ~]# cat /proc/3986/cmdline
/usr/libexec/qemu-kvm-nameguest=instance-00001782,debug-threads=on-S-objectsecret,id=masterKey0,format=raw,file=/var/lib/libvirt/qemu/domain-1-instance-00001782/master-key.aes-machinepc-i440fx-rhel7.3.0,accel=kvm,usb=off-cpuSandyBridge,+vme,+ds,+acpi,+ss,+ht,+tm,+pbe,+dtes64,+monitor,+ds_cpl,+vmx,+smx,+est,+tm2,+xtpr,+pdcm,+pcid,+dca,+osxsave,+arat,+xsaveopt,+pdpe1gb-m16384-realtimemlock=off-smp8,sockets=8,cores=1,threads=1-uuid6f2ade4b-2732-4202-b750-48daee771b06-smbiostype=1,manufacturer=Red Hat,product=OpenStack Compute,version=14.0.3-9.el7ost,serial=ce409aed-edba-4f0f-a40e-3bbb5566f414,uuid=6f2ade4b-2732-4202-b750-48daee771b06,family=Virtual Machine-no-user-config-nodefaults-chardevsocket,id=charmonitor,path=/var/lib/libvirt/qemu/domain-1-instance-00001782/monitor.sock,server,nowait-monchardev=charmonitor,id=monitor,mode=control-rtcbase=utc,driftfix=slew-globalkvm-pit.lost_tick_policy=discard-no-hpet-no-shutdown-bootstrict=on-devicepiix3-usb-uhci,id=usb,bus=pci.0,addr=0x1.0x2-drivefile=/var/lib/nova/instances/6f2ade4b-2732-4202-b750-48daee771b06/disk,format=qcow2,if=none,id=drive-virtio-disk0,cache=none,aio=native-devicevirtio-blk-pci,scsi=off,bus=pci.0,addr=0x4,drive=drive-virtio-disk0,id=virtio-disk0,bootindex=1-netdevtap,fd=28,id=hostnet0,vhost=on,vhostfd=30-devicevirtio-net-pci,netdev=hostnet0,id=net0,mac=fa:16:3e:02:89:51,bus=pci.0,addr=0x3-add-fdset=2,fd=32-chardevfile,id=charserial0,path=/dev/fdset/2,append=on-deviceisa-serial,chardev=charserial0,id=serial0-chardevpty,id=charserial1-deviceisa-serial,chardev=charserial1,id=serial1-deviceusb-tablet,id=input0,bus=usb.0,port=1-vnc0.0.0.0:0-ken-us-devicecirrus-vga,id=video0,bus=pci.0,addr=0x2-devicevirtio-balloon-pci,id=balloon0,bus=pci.0,addr=0x5-msgtimestamp=on
```

python的实现可以这样，先也是去listdomain，可以知道domain name和uuid

```python
import libvirt
import libvirt_qemu
conn = libvirt.open('qemu:///system')
conn.listAllDomains(0)
[<libvirt.virDomain object at 0x25867d0>, <libvirt.virDomain object at 0x2586a50>, <libvirt.virDomain object at 0x25868d0>]

domain = conn.listAllDomains(0)[0]
domain.name()
'instance-0000183e'
domain.UUIDString()
'a3bd4172-eb8b-4f0c-b31d-735cf0a71c61'
```

我们知道一个kvm vm其实就是一个qemu-kvm进程，那我们其实不需要去遍历/proc目录，只需要这样就可以取到pid

```python
from subprocess import check_output
check_output(['pidof', 'qemu-kvm'])
'264742 251524 3986\n'

pids = check_output(['pidof', 'qemu-kvm'])
pids.split(' ')
['264742', '251524', '3986\n']
```

而三个pid跟三个domain的对应关系，就可以参考上面的，看下cmdline的内容就知道了。

## list task

那我们知道，进程是包含线程的，我们需要把这个信息找出来。在`/prod/pid`下存在一个task目录，task目录下存在`task/tid`这样的目录，就是每个进程中每个线程的信息。

kvmtop中的实现：

```go
// get core thread IDs
vCPUThreads, err := libvirtDomain.QemuMonitorCommand("info cpus", libvirt.DOMAIN_QEMU_MONITOR_COMMAND_HMP)

regThreadID := regexp.MustCompile("thread_id=([0-9]*)\\s")

	// get thread IDs
	tasksFolder := fmt.Sprint(config.Options.ProcFS, "/", domain.PID, "/task/*")
	files, err := filepath.Glob(tasksFolder)
	if err != nil {
		return
	}
	otherThreadIDs := make([]int, 0)
	i := 0
	for _, f := range files {
		taskID, _ := strconv.Atoi(path.Base(f))
		found := false
		for _, n := range coreThreadIDs {
			if taskID == n {
				// taskID is for vCPU core. skip.
				found = true
				break
			}
		}
		if found {
			// taskID is for vCPU core. skip.
			continue
		}
		// taskID is not for a vCPU core
		otherThreadIDs = append(otherThreadIDs, taskID)
		oldThreadIds = removeFromArray(oldThreadIds, taskID)
		i++
	}
```

这里把qemu-kvm中的线程分成了两个部分：

* core thread：VCORES from threadIDs
* others：stats for other threads (i/o or emulation)

哪些算是core的呢？

比如我这台vm有8个cpu（| flavor                               | 8vcpu.16mem.80ssd.0eph (6c296db4-8c83-42cf-a8e4-97a497697a48)        |）

通过qemumonitorcommand，`info cpus`，可以获取到这8个cpu所用到的线程号。然后对照/proc/pid/task下的目录，剩下的就是others

python中可以这样

```python
libvirt_qemu.qemuMonitorCommand(domain, 'info cpus', 1)
'* CPU #0: pc=0xffffffff9f16ae9b (halted) thread_id=4031\r\n  CPU #1: pc=0xffffffff9f16ae9b (halted) thread_id=4032\r\n  CPU #2: pc=0xffffffff9f16ae9b (halted) thread_id=4037\r\n  CPU #3: pc=0xffffffff9f16ae9b (halted) thread_id=4038\r\n  CPU #4: pc=0xffffffff9f16ae9b (halted) thread_id=4039\r\n  CPU #5: pc=0xffffffff9f16ae9b (halted) thread_id=4040\r\n  CPU #6: pc=0xffffffff9f16ae9b (halted) thread_id=4041\r\n  CPU #7: pc=0xffffffff9f16ae9b (halted) thread_id=4042\r\n'

threads = libvirt_qemu.qemuMonitorCommand(domain, 'info cpus', 1)
re.findall(r'thread_id=(.+)\r', threads)
['4031', '4032', '4037', '4038', '4039', '4040', '4041', '4042']

os.listdir('/proc/3986/task')
['3986', '4023', '4031', '4032', '4037', '4038', '4039', '4040', '4041', '4042', '4049']
```

['4031', '4032', '4037', '4038', '4039', '4040', '4041', '4042'] —— 这就是core thread

['3986', '4023', '4049'] —— 就是others

## 计算steal time

最后这一步，就是对core thread & others，分别计算steal time，这个steal time怎么算的呢？

`/proc/pid/schedstat`可以取到time spent waiting on a runqueue，这个值可以被认为是进程等待runqueue所花费的时间，如果花费的时间过长，那么就说明自己的cpu time被stolen了。

kvmtop的实现：

```go
stats := ProcPIDSchedStat{PID: pid}
filepath := fmt.Sprint(config.Options.ProcFS, "/", strconv.Itoa(pid), "/schedstat")
filecontent, _ := ioutil.ReadFile(filepath)

_, err := fmt.Fscan(
   bytes.NewBuffer(filecontent),
   &stats.Cputime,
   &stats.Runqueue,
   &stats.Timeslices,
)
```

取到之后，再相加并求平均值

```go
func CpuPrintThreadMetric(domain *models.Domain, lookupMetric string, metric string) string {
   threadIDs := domain.GetMetricIntArray(lookupMetric)
   var measurementSum float64
   var measurementCount int
   for _, threadID := range threadIDs {
      metricName := fmt.Sprint(metric, "_", threadID)
      measurementStr := domain.GetMetricDiffUint64(metricName, true)
      if measurementStr == "" {
         continue
      }
      measurement, err := strconv.ParseUint(measurementStr, 10, 64)
      if err != nil {
         continue
      }
      measurementSeconds := float64(measurement) / 1000000000 // since counters are nanoseconds
      measurementSum += measurementSeconds
      measurementCount++
   }

   var avg float64
   if measurementCount > 0 {
      avg = float64(measurementSum) / float64(measurementCount)
   }
   percent := avg * 100
   return fmt.Sprintf("%.0f", percent)
}
```

而由于这个值在系统中是nanoseconds十亿分之一秒，所以还要/ 1000000000得到seconds

python的实现也就是去读一下文件，计算就好了

```python
f = open('/proc/4031/schedstat')
metrics = f.readline()
metrics
'189601784545 36269883 2783129\n'

metrics.split(' ')
['189601784545', '36269883', '2783129\n']
```

# References

[1] [https://github.com/cha87de/kvmtop]( https://github.com/cha87de/kvmtop)

