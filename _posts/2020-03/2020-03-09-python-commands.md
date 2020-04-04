---
layout: post
title:  "python2的commands和python3的subprocess"
date:   2020-03-09 23:01:00 +0800
categories: Python
tags: Python-Language
excerpt: python2的commands和python3的subprocess
mathjax: true
typora-root-url: ../
---

# python2 commands

想要在python中得到shell命令输出的结果，Python2里有个很好用的commands

```python
>>> import commands
>>> status, output = commands.getstatusoutput("openssl s_client -servername ci22sy-keystone-admin.webex.com -connect ci22sy-keystone-admin.webex.com:443 2>/dev/null | openssl x509 -enddate -noout 2>/dev/null |awk -F '=' '{print $2}'")
KeyboardInterrupt
>>> print(status)
0
>>> print(output)
Jan  2 17:29:00 2021 GMT
```

# python3 subprocess

然后在python3里面，这个好用的commands不见了

据说subprocess可以代替一下，简单的命令比如这样

```python
>>> import subprocess
>>> status, output = subprocess.getstatusoutput("pwd")
>>> print(status)
0
>>> print(output)
/
```

可是如果试试刚刚那个命令，就不行了

```python
>>> status, output = subprocess.getstatusoutput("openssl s_client -servername ci22sy-keystone-admin.webex.com -connect ci22sy-keystone-admin.webex.com:443 2>/dev/null | openssl x509 -enddate -noout 2>/dev/null |awk -F '=' '{print $2}'")
>>> print(status)
0
>>> print(output)

```

原因是这里有管道pipeline的存在

```shell
Replacing shell pipeline
output=`dmesg | grep hda`
becomes:

p1 = Popen(["dmesg"], stdout=PIPE)
p2 = Popen(["grep", "hda"], stdin=p1.stdout, stdout=PIPE)
p1.stdout.close()  # Allow p1 to receive a SIGPIPE if p2 exits.
output = p2.communicate()[0]
The p1.stdout.close() call after starting the p2 is important in order for p1 to receive a SIGPIPE if p2 exits before p1.

Alternatively, for trusted input, the shell’s own pipeline support may still be used directly:

output=`dmesg | grep hda`
becomes:

output=check_output("dmesg | grep hda", shell=True)
```

所以可以这样

```python
>>> import subprocess
>>> output=subprocess.check_output("/usr/include/openssl s_client -servername ci22sy-keystone-admin.webex.com -connect ci22sy-keystone-admin.webex.com:443 2>/dev/null | /usr/include/openssl x509 -enddate -noout 2>/dev/null", shell=True)
>>> print(output)
b'notAfter=Jan 2 17:29:00 2021 GMT\n'
```

