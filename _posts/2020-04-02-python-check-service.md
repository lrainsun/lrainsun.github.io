---
layout: post
title:  "python检查服务是否正常运行"
date:   2020-04-02 23:00:00 +0800
categories: Python
tags: Python-Language
excerpt: python检查服务是否正常运行
mathjax: true
typora-root-url: ../
---

# python检查服务是否正常运行

* 检查某个服务是不是在运行

```python
def check_service_running(service):
    return_code = os.system('systemctl status ' + service)
    if return_code == 0:
        return "service is running"
    if return_code == 768:
        return "service is not running"
    if return_code == 1024:
        return "service not exist"
```

* 检查进程是不是在运行

```python
def check_process_running(process):
    for proc in psutil.process_iter():
        if proc.name().lower() == process.lower():
            return "process is running"
    return "process is not running"
```

* 检查端口是否被使用

```python
def check_port_used(port):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    try:
        s.connect(('127.0.0.1', port))
        return "port used"
    except OSError:
        return "port not used"
    finally:
        s.close()
```

