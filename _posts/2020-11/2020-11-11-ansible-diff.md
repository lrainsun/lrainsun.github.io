---
layout: post
title:  "Ansible版本差异"
date:   2020-11-11 23:00:00 +0800
categories: Airflow
tags: Airflow-Tutorial
excerpt: Ansible版本差异
mathjax: true
typora-root-url: ../
---

# Ansible版本差异

从ansible版本2.4.2升到2.10，playbook会有一些差异，不支持的地方

## version_compare

version_compare在2.10被废弃了，改成了version

2.4.2原来的写法是这样的：

```
version_compare('defined_kernel','<')
```

修改后

```
version(defined_kernel,'<')
```

## |changed， |failed

2.4.2里，过滤都是用|

比如`|changed，|failed`

新版本里是`.changed, .failed`

## register

```yaml
- name: Create journal file
  file: name=/var/log/journal state=directory owner=root group=root
  register: journallog

- name: restart systemd-journald if log file changed
  service: name=systemd-journald state=restarted
  ignore_errors: True
  register: journal
  when: journallog.changed

- name: restart again if first restart failed
  service: name=systemd-journald state=restarted
  when: journal|failed
```

这个写法在2.4.2是没问题的，journal总会有failed

新版本里会提示错误

```
"msg": "The conditional check 'journal is defined and journal.failed' failed. The error was: error while evaluating conditional (journal is defined and journal.failed): 'dict object' has no attribute 'failed'\n\nThe error appears to be in '/tmp/ocp_ansible_module/roles/common/tasks/main.yml': line 161, column 3, but may\nbe elsewhere in the file depending on the exact syntax problem.\n\nThe offending line appears to be:\n\n\n- name: restart again if first restart failed\n  ^ here\n",
```

本来理解是上面的when: journallog.changed如果判断是失败的，那么journal就不应该被定义，而其实，不管是否执行，都会有journal

journallog.changed是false的时候

```json
    "journal": {
        "changed": false,
        "skip_reason": "Conditional result was False",
        "skipped": true
    }
```

journallog.changed是true的时候

```json
    "journal": {
        "changed": true,
        "failed": false,
        "name": "systemd-journald",
        "state": "started",
        "status": {
            "ActiveEnterTimestamp": "Wed 2020-11-11 09:28:05 GMT",
            "ActiveEnterTimestampMonotonic": "1100094977347",
            。。。。
    }
```









