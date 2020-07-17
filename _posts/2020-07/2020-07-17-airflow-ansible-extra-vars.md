---
layout: post
title:  "Ansible Python API extra vars"
date:   2020-07-17 23:00:00 +0800
categories: Airflow
tags: Airflow-Tutorial
excerpt: Ansible Python API extra vars
mathjax: true
typora-root-url: ../
---

# Ansible Python API extra vars

之前有试过传extra_vars，相当于跑ansible-playbook的时候的-e，可以输入key=value的值

类似这样：

```python
rbt.run_playbook(playbook_path='/Users/minsu/Documents/minsu/ansible/ocp_ansible_module/0_common.yml', extra_vars={"ansible_sudo_pass": "", "@": "dd"})
```

但其实-e还可以传文件的

> -e EXTRA_VARS, --extra-vars=EXTRA_VARS set additional variables as key=value or YAML/JSON, if filename prepend with @

这时候我们要怎么传入呢？

需要在一开始初始化的时候传入，类似这样：

```python
context.CLIARGS = ImmutableDict(connection='smart', module_path=[self.module_path], roles_path=self.roles_path, forks=10, become=True, private_key_file='id_rsa', become_method='sudo', become_user='root', become_ask_pass=False, check=False, diff=False, verbosity=4, syntax=None, start_at_task=None, extra_vars={"@/etc/kolla/passwords.yml"})
```

在context.CLIARGS定义的extra_vars，如果是文件，会被解析，并且把文件里的内容解析成dict

```python
def load_extra_vars(loader):
    extra_vars = {}
    for extra_vars_opt in context.CLIARGS.get('extra_vars', tuple()):
        data = None
        extra_vars_opt = to_text(extra_vars_opt, errors='surrogate_or_strict')
        if extra_vars_opt.startswith(u"@"):
            # Argument is a YAML file (JSON is a subset of YAML)
            data = loader.load_from_file(extra_vars_opt[1:])
        elif extra_vars_opt and extra_vars_opt[0] in u'[{':
            # Arguments as YAML
            data = loader.load(extra_vars_opt)
        else:
            # Arguments as Key-value
            data = parse_kv(extra_vars_opt)

        if isinstance(data, MutableMapping):
            extra_vars = combine_vars(extra_vars, data)
        else:
            raise AnsibleOptionsError("Invalid extra vars data supplied. '%s' could not be made into a dictionary" % extra_vars_opt)

    return extra_vars
```

而像我之前在run_playbook时传入的extra_vars是不会被load，像下面这样

```python
    def run_playbook(self, playbook_path, extra_vars=None):
        self.playbook_path = playbook_path
        import traceback
        try:
            self.callback = PlayBookResultsCollector()
            if extra_vars:
                self.variable_manager._extra_vars = extra_vars

            executor = PlaybookExecutor(
                playbooks=[playbook_path], inventory=self.inventory, variable_manager=self.variable_manager,
                loader=self.loader,
                passwords=None,
            )
```

如果如果是传文件，一定要在最前面定义，而且，所有的extra_vars都要放在一起定义，否则会被后面的覆盖掉

