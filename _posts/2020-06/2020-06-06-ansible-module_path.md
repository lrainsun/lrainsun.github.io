---
layout: post
title:  "Ansible module path"
date:   2020-06-06 23:00:00 +0800
categories: Airflow
tags: Airflow-Tutorial
excerpt: Ansible module path
mathjax: true
typora-root-url: ../
---

# Ansible module path

一个折腾了我好久的问题

在跑一个ansible role的时候，需要用到我们自定义的module，放在ocp_ansible_module/library下了，默认的值如下

```yaml
DEFAULT_MODULE_PATH:
  name: Modules Path
  description: Colon separated paths in which Ansible will search for Modules.
  default: ~/.ansible/plugins/modules:/usr/share/ansible/plugins/modules
  env: [{name: ANSIBLE_LIBRARY}]
  ini:
  - {key: library, section: defaults}
  type: pathspec
```

我们的位置不在默认的里面

有一个办法是可以在ansible.cfg里定义

```ini
library        = ocp_ansible_module/library/
```

可是我希望在代码里传入，而不是写死在config文件里

按理说已经传入了

```python
context.CLIARGS = ImmutableDict(connection='smart', module_path=[self.module_path],
```

可是不知为啥不生效，还是一直报错，后来发现，我们跑role的时候，是这样的

```python
        play = Play().load(play_source, variable_manager=self.variable_manager, loader=self.loader)
        tqm = None
        self.callback = PlayBookResultsCollector()
        import traceback
        try:
            if extra_vars:
                self.variable_manager._extra_vars = extra_vars
            tqm = TaskQueueManager(
                inventory=self.inventory,
                variable_manager=self.variable_manager,
                loader=self.loader,
                passwords=None,
                stdout_callback="minimal",
            )
            tqm._stdout_callback = self.callback
            constants.HOST_KEY_CHECKING = False
            tqm.run(play)
```

查看了代码，module_path会在TaskQueueManager初始化的时候读取

```python
        # make sure any module paths (if specified) are added to the module_loader
        if context.CLIARGS.get('module_path', False):
            print("get module path")
            for path in context.CLIARGS['module_path']:
                if path:
                    module_loader.add_directory(path)
```

而我们的play定义放在了TaskQueueManager初始化之前，这就有问题了，所以解决办法很简单，放到TaskQueueManager初始化之后就好了。。。折腾了好久

```python
				tqm = None
        self.callback = PlayBookResultsCollector()
        import traceback
        try:
            if extra_vars:
                self.variable_manager._extra_vars = extra_vars
            tqm = TaskQueueManager(
                inventory=self.inventory,
                variable_manager=self.variable_manager,
                loader=self.loader,
                passwords=None,
                stdout_callback="minimal",
            )
            tqm._stdout_callback = self.callback
            constants.HOST_KEY_CHECKING = False
            play = Play().load(play_source, variable_manager=self.variable_manager, loader=self.loader)
            tqm.run(play)
```

