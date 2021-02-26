---
layout: post
title:  "Ansible Python API初探"
date:   2020-05-28 23:00:00 +0800
categories: Ansible
tags: Ansible-pythonapi
excerpt: Ansible API初探
mathjax: true
typora-root-url: ../
---

# Ansible Python API初探

ansible在跑完playbook之后，我们要怎么知道，多少host成功，多少失败了呢？想要把run ansible的动作放到airflow上做自动化处理的话，获取运行结果这个就很必要了。

今天看了一下ansible api，做了一个小小的尝试。

```python
from ansible import constants
from collections import namedtuple
from ansible import context
from ansible.module_utils.common.collections import ImmutableDict
from ansible.parsing.dataloader import DataLoader
from ansible.playbook.play import Play
from ansible.executor.task_queue_manager import TaskQueueManager
from ansible.executor.playbook_executor import PlaybookExecutor
from ansible.plugins.callback import CallbackBase
from ansible.inventory.manager import InventoryManager
from ansible.vars.manager import VariableManager
import json

class ModelResultsCollector(CallbackBase):
    def __init__(self, *args, **kwargs):
        super(ModelResultsCollector, self).__init__(*args, **kwargs)
        self.host_ok = {}
        self.host_unreachable = {}
        self.host_failed = {}

    def v2_runner_on_unreachable(self, result):
        self.host_unreachable[result._host.get_name()] = result

    def v2_runner_on_ok(self, result, *args, **kwargs):
        self.host_ok[result._host.get_name()] = result

    def v2_runner_on_failed(self, result, *args, **kwargs):
        self.host_failed[result._host.get_name()] = result


class PlayBookResultsCollector(CallbackBase):
    CALLBACK_VERSION = 2.0

    def __init__(self, *args, **kwargs):
        super(PlayBookResultsCollector, self).__init__(*args, **kwargs)
        self.task_ok = {}
        self.task_skipped = {}
        self.task_failed = {}
        self.task_status = {}
        self.task_unreachable = {}

    def v2_runner_on_ok(self, result, *args, **kwargs):
        self.task_ok[result._host.get_name()] = result

    def v2_runner_on_failed(self, result, *args, **kwargs):
        self.task_failed[result._host.get_name()] = result

    def v2_runner_on_unreachable(self, result):
        self.task_unreachable[result._host.get_name()] = result

    def v2_runner_on_skipped(self, result):
        self.task_ok[result._host.get_name()] = result

    def v2_playbook_on_stats(self, stats):
        hosts = sorted(stats.processed.keys())
        for h in hosts:
            t = stats.summarize(h)
            self.task_status[h] = {
                "ok": t['ok'],
                "changed": t['changed'],
                "unreachable": t['unreachable'],
                "skipped": t['skipped'],
                "failed": t['failures']
            }


class AnsibleRunner(object):
    """
    This is a General object for parallel execute modules.
    """

    def __init__(self, ips=None, *args, **kwargs):
        self.ips = ips
        self.inventory = None
        self.variable_manager = None
        self.loader = None
        self.options = None
        self.passwords = None
        self.callback = None
        self.__initializeData()
        self.results_raw = {}

    def __initializeData(self):
        context.CLIARGS = ImmutableDict(connection='smart', module_path=['/mymodules'], forks=10, become=None, become_method=None, become_user=None, check=False, diff=False, verbosity=4)

        self.loader = DataLoader()
        self.inventory = InventoryManager(loader=self.loader, sources='/Users/minsu/Documents/minsu/ansible/inventory')
        self.variable_manager = VariableManager(loader=self.loader, inventory=self.inventory)
        print(self.variable_manager.get_vars())

    def run_model(self, module_name, module_args):
        """
        run module from andible ad-hoc.
        module_name: ansible module_name
        module_args: ansible module args
        """
        play_source = dict(
            name="Ansible Play",
            hosts=self.ips,
            gather_facts='no',
            tasks=[dict(action=dict(module=module_name, args=module_args))]
        )

        play = Play().load(play_source, variable_manager=self.variable_manager, loader=self.loader)
        tqm = None
        self.callback = ModelResultsCollector()
        import traceback
        try:
            tqm = TaskQueueManager(
                inventory=self.inventory,
                variable_manager=self.variable_manager,
                loader=self.loader,
                passwords=self.passwords,
                stdout_callback="minimal",
            )
            tqm._stdout_callback = self.callback
            constants.HOST_KEY_CHECKING = False 
            tqm.run(play)
        except Exception as err:
            print(traceback.print_exc())
        finally:
            if tqm is not None:
                tqm.cleanup()

    def get_model_result(self):
        self.results_raw = {'success': {}, 'failed': {}, 'unreachable': {}}
        for host, result in self.callback.host_ok.items():
            hostvisiable = host.replace('.', '_')
            self.results_raw['success'][hostvisiable] = result._result

        for host, result in self.callback.host_failed.items():
            hostvisiable = host.replace('.', '_')
            self.results_raw['failed'][hostvisiable] = result._result

        for host, result in self.callback.host_unreachable.items():
            hostvisiable = host.replace('.', '_')
            self.results_raw['unreachable'][hostvisiable] = result._result

        return self.results_raw

    def get_playbook_result(self):
        self.results_raw = {'skipped': {}, 'failed': {}, 'ok': {}, "status": {}, 'unreachable': {}, "changed": {}}
        for host, result in self.callback.task_ok.items():
            self.results_raw['ok'][host] = result._result

        for host, result in self.callback.task_failed.items():
            self.results_raw['failed'][host] = result._result

        for host, result in self.callback.task_status.items():
            self.results_raw['status'][host] = result

        for host, result in self.callback.task_skipped.items():
            self.results_raw['skipped'][host] = result._result

        for host, result in self.callback.task_unreachable.items():
            self.results_raw['unreachable'][host] = result._result
        return self.results_raw


if __name__ == '__main__':
    host="10.225.28.32"
    rbt = AnsibleRunner(host)
    rbt.run_model('shell','cat /etc/redhat-release')
    result = json.dumps(rbt.get_model_result(),indent=4)
    print(result)
```

inventory定义

```shell
(ansible) MINSU-M-M1RW:ansible minsu$ cat /Users/minsu/Documents/minsu/ansible/inventory
[host]
10.225.28.32 ansible_ssh_user=ocp ansible_ssh_pass='********'
```

其实就是通过ansible对传入的主机`10.225.28.32`跑了一个shell module，命令是`cat /etc/redhat-release`，然后把返回结果打印出来了

```json
{
    "success": {
        "10_225_28_32": {
            "changed": true,
            "end": "2020-05-28 11:39:25.881297",
            "stdout": "CentOS Linux release 7.4.1708 (Core) ",
            "cmd": "cat /etc/redhat-release",
            "rc": 0,
            "start": "2020-05-28 11:39:25.875846",
            "stderr": "",
            "delta": "0:00:00.005451",
            "invocation": {
                "module_args": {
                    "creates": null,
                    "executable": null,
                    "_uses_shell": true,
                    "strip_empty_ends": true,
                    "_raw_params": "cat /etc/redhat-release",
                    "removes": null,
                    "argv": null,
                    "warn": true,
                    "chdir": null,
                    "stdin_add_newline": true,
                    "stdin": null
                }
            },
            "stdout_lines": [
                "CentOS Linux release 7.4.1708 (Core) "
            ],
            "stderr_lines": [],
            "ansible_facts": {
                "discovered_interpreter_python": "/usr/bin/python"
            },
            "_ansible_no_log": false
        }
    },
    "failed": {},
    "unreachable": {}
}
```



