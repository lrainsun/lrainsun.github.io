---
layout: post
title:  "PXE & IPXE"
date:   2020-01-08 17:00:00 +0800
categories: Openstack
tags: Openstack-Ironic
excerpt: PXE & IPXE
mathjax: true
typora-root-url: ../
---

# PXE & IPXE

在安装standalone的ironic的时候，发现会起两个container，一个ironic_pxe，一个ironic_ipxe。原因是我在global.yml里设置了`enable_ironic_ipxe: "yes"`

```yaml
  ironic-pxe:
    container_name: ironic_pxe
    group: ironic-pxe
    enabled: true
    image: "{{ ironic_pxe_image_full }}"
    volumes: "{{ ironic_pxe_default_volumes + ironic_pxe_extra_volumes }}"
    dimensions: "{{ ironic_pxe_dimensions }}"
  ironic-ipxe:
    container_name: ironic_ipxe
    group: ironic-ipxe
    enabled: "{{ enable_ironic_ipxe | bool }}"
    image: "{{ ironic_pxe_image_full }}"
    volumes: "{{ ironic_ipxe_default_volumes + ironic_ipxe_extra_volumes }}"
    dimensions: "{{ ironic_ipxe_dimensions }}"
```

看起来PXE是默认启动的，而IPXE是需要enable的，那PXE & IPXE两者是有什么区别呢？

PXE我们应该比较熟悉，用的比较多，而IPXE应该是PXE的扩展版

# PXE

Preboot Execution Environment (PXE): The PXE enables system’s BIOS and network interface card (NIC) to bootstrap a computer from the network in place of a disk. 由Intel开发，工作于client/server网络模式，支持工作站通过网络从远端服务器下载镜像，并由此进行网络启动和安装。

![image-20200108154715599](/../assets/images/image-20200108154715599.png)

- 服务器发送dhcp请求
- dhcp-server 响应dhcp请求，分配ip地址，并且返回next-server（tftp服务器地址）和filename（一般为pxelinux.0）
- 服务器获得ip地址，然后通过tftp协议从next-server下载pxelinux.0
- pxelinux.0读取tftp-server上pxelinux.cfg目录下的配置
- pxelinux.0展示界面，根据用户输入的选项进行安装或者启动

# IPXE

IPXE是PXE的一个开源实现，通过IPXE能让网卡直接支持网络启动，而不依赖于网卡自带的PXE固件。同时相比PXE，IPXE支持更多的协议。传统的PXE只能通过TFTP进行传输，而IPXE支持HTTP，ISCSI和ATA over Ethernet（AoE），因此传输速率大大提升。

- 服务器 **第一次** 发送dhcp请求
- dhcp-server 响应dhcp请求，分配ip地址，并且返回next-server（tftp服务器地址）和filename（一般为undionly.kpxe）
- 服务器通过tftp协议下载unidonly.kpxe文件
- undionly.kpxe **第二次** 发送dhcp请求
- dhcp-server 响应dhcp请求，分配ip地址，并且返回一个URL
- 服务器访问URL，并将硬件信息（如sn）作为参数传递，然后获取根据sn生成的ipxe脚本
- 服务器执行ipxe脚本，通过http协议下载操作系统kernel和initrd
- 服务器利用kernel和initrd正常引导

