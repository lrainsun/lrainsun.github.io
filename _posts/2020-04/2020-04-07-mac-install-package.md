---
layout: post
title:  "MAC安装openstack ironicclient"
date:   2020-04-06 23:00:00 +0800
categories: Openstack
tags: Openstack-Ironic
excerpt: MAC安装openstack ironicclient
mathjax: true
typora-root-url: ../
---

# MAC安装包错误

嗯，有一丢丢无聊的blog

今天想在mac上`pip install ironic`的时候，死活装不上，虽然后来发现其实并没有必要装。。但是当下非常之纠结，提示

```shell
  ERROR: Command errored out with exit status 1:
   command: /usr/local/opt/python@2/bin/python2.7 -u -c 'import sys, setuptools, tokenize; sys.argv[0] = '"'"'/private/var/folders/nf/ng8ttnm16vn5z53bgcvp4bnw0000gn/T/pip-install-YF2JgA/greenlet/setup.py'"'"'; __file__='"'"'/private/var/folders/nf/ng8ttnm16vn5z53bgcvp4bnw0000gn/T/pip-install-YF2JgA/greenlet/setup.py'"'"';f=getattr(tokenize, '"'"'open'"'"', open)(__file__);code=f.read().replace('"'"'\r\n'"'"', '"'"'\n'"'"');f.close();exec(compile(code, __file__, '"'"'exec'"'"'))' bdist_wheel -d /private/var/folders/nf/ng8ttnm16vn5z53bgcvp4bnw0000gn/T/pip-wheel-QlS0L9
       cwd: /private/var/folders/nf/ng8ttnm16vn5z53bgcvp4bnw0000gn/T/pip-install-YF2JgA/greenlet/
  Complete output (9 lines):
  running bdist_wheel
  running build
  running build_ext
  building 'greenlet' extension
  creating build
  creating build/temp.macosx-10.12-x86_64-2.7
  clang -fno-strict-aliasing -fno-common -dynamic -g -O2 -DNDEBUG -g -fwrapv -O3 -Wall -Wstrict-prototypes -I/usr/local/include -I/usr/local/opt/openssl/include -I/usr/local/opt/sqlite/include -I/usr/local/Cellar/python@2/2.7.15_3/Frameworks/Python.framework/Versions/2.7/include/python2.7 -c greenlet.c -o build/temp.macosx-10.12-x86_64-2.7/greenlet.o
  xcrun: error: invalid active developer path (/Library/Developer/CommandLineTools), missing xcrun at: /Library/Developer/CommandLineTools/usr/bin/xcrun
  error: command 'clang' failed with exit status 1
  ----------------------------------------
  ERROR: Failed building wheel for greenlet
```

`missing xcrun at: /Library/Developer/CommandLineTools/usr/bin/xcrun` google了一下是需要安装Command Line Tools，这样

```shell
MINSU-M-M1RW:minsu minsu$ xcode-select --install
xcode-select: note: install requested for command line developer tools
```

然后会弹出来一个软件安装框提示安装，可是几个小时都没反应

所以可以这样，直接去网站下载对应版本的Command Line Tools：

[https://developer.apple.com/download/more/](https://developer.apple.com/download/more/)

![image-20200407142058817](/../assets/images/image-20200407142058817.png)

下载安装之后，就可以了，再次安装ironic就成功了

# Ironic Client Command-Line Interface

哦，其实我只是想要在mac端装个Ironic Client Command-Line Interface (CLI)，并没有必要安装ironic，只需要安装python-ironicclient

但貌似最新版本的`python-ironicclient`没有ironic那个可执行文件了，openstackclient（5.2.0）也是

最后安装了指定版本，我们现在在用的1.7.1

```shell
bash-3.2$ pip install python-ironicclient==1.7.1

bash-3.2$ ironic --version
1.7.1

bash-3.2$ which ironic
/usr/local/bin/ironic
```

然后其实Ironic Client Command-Line在openstackclient里也有，可是直接`pip install python-openstackclient`装出来的不行

```shell
bash-3.2$
openstack: 'baremetal list' is not an openstack command. See 'openstack --help'.
Did you mean one of these?
  credential create
  credential delete
  credential list
  credential set
  credential show
```

if python >=2.7.13, use `pip2` instead.

```shell
brew install python
pip2 install --upgrade pip setuptools
pip2 install --upgrade python-openstackclient
```

```shell
bash-3.2$ openstack baremetal node list
+--------------------------------------+----------------+--------------------------------------+-------------+--------------------+-------------+
| UUID                                 | Name           | Instance UUID                        | Power State | Provisioning State | Maintenance |
+--------------------------------------+----------------+--------------------------------------+-------------+--------------------+-------------+
| f62257bf-ca05-4efa-b6f2-75777dc8016c | 10.240.194.48  | c865c2ca-30f5-44cd-a866-b649eea9192c | power on    | active             | False       |
| b2563c20-55ad-4c2a-938e-b9a0785b6692 | 10.240.194.55  | 2408b357-279b-4f7d-aa3f-d3568e0fd1a7 | power on    | active             | False       |
| f63cf06e-ae23-4e16-9d82-a48d3e987e7a | 10.240.194.57  | a44ab96d-b4ec-41ca-a8a2-c6ac2fd3c2db | power on    | active             | False       |
```

# References

[1] [https://gist.github.com/gildas/351af1ee2e64163f6d29](https://gist.github.com/gildas/351af1ee2e64163f6d29)

