---
layout: post
title:  "Ansible dynamic inventory"
date:   2021-03-01 23:00:00 +0800
categories: Ansible
tags: Ansible
excerpt: Ansible dynamic inventory
mathjax: true
typora-root-url: ../
---

# Ansible dynamic inventory

动态 Inventory 指通过外部脚本获取主机列表，并按照 ansible 所要求的格式返回给 ansilbe 命令的。比如我们可以传一个python script，只不过对于返回值是有格式要求的，比如是json文件

用于生成 JSON 的脚本**对实现语言没有要求**，它可以是一个可执行脚本、二进制文件，或者其他任何可以运行文件，但是输出必须为 JSON 格式

同时必须能够支持两个参数`--list`和`--host <hostname>`

* list：用于返回所有的主机组信息，每个组所包含的主机列表、所含子组列表、主机组变量列表都应该是字典形式的，用来存放主机变量

  * ```yaml
    {
        "group001": {
            "hosts": ["host001", "host002"],
            "vars": {
                "var1": true
            },
            "children": ["group002"]
        },
        "group002": {
            "hosts": ["host003","host004"],
            "vars": {
                "var2": 500
            },
            "children":[]
        }
    
    }
    ```

* host：返回指定主机的变量列表，或者返回一个空的字典

  * ```yaml
    {
        "VAR001": "VALUE",
        "VAR002": "VALUE",
    }
    ```

然而这样比较没有效率，因为ansible会对每一个host调用一遍--host

所以当在--list返回的最上层是_meta时，约定脚本就会输出所有host的variables，那么ansible就不需要再去为每个host调用一遍--host，这样就提升了效率

```yaml
{

    # results of inventory script as above go here
    # ...

    "_meta": {
        "hostvars": {
            "host001": {
                "var001" : "value"
            },
            "host002": {
                "var002": "value"
            }
        }
    }
}
```

