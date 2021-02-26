---
layout: post
title:  "Ansible Python API run playbook"
date:   2020-05-29 23:00:00 +0800
categories: Ansible
tags: Ansible-pythonapi
excerpt: Ansible Python API run playbook
mathjax: true
typora-root-url: ../
---

# Ansible Python API run playbook

昨天试了通过ansible python api去run一个module，今天尝试了run playbook

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
from ansible.plugins.callback.default import CallbackModule
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
        print(result.__dict__)

    def v2_runner_on_failed(self, result, *args, **kwargs):
        self.task_failed[result._host.get_name()] = result
        print(result.__dict__)

    def v2_runner_on_unreachable(self, result):
        self.task_unreachable[result._host.get_name()] = result
        print(result.__dict__)

    def v2_runner_on_skipped(self, result):
        self.task_ok[result._host.get_name()] = result
        print(result.__dict__)

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

    def __init__(self, hosts=None, module_path='', inventory='', ssh_pass='', *args, **kwargs):
        self.hosts = hosts
        self.inventory = None
        self.variable_manager = None
        self.loader = None
        self.inventory = inventory
        self.ssh_pass = ssh_pass
        self.callback = None
        self.module_path = module_path
        self.__initializeData()
        self.results_raw = {}

    def __initializeData(self):
        context.CLIARGS = ImmutableDict(connection='smart', module_path=[self.module_path], forks=10, become=True, become_method='sudo', become_user='root', become_ask_pass=False, check=False, diff=False, verbosity=4, syntax=None, start_at_task=None, config_file='/Users/minsu/Documents/minsu/ansible/ansible.cfg')

        self.loader = DataLoader()
        self.inventory = InventoryManager(loader=self.loader, sources=self.inventory)
        self.inventory.subset(['ci81hf1cmp003','ci81hf1cmp001','ci81hf1cmp002'])
        self.variable_manager = VariableManager(loader=self.loader, inventory=self.inventory)
        self.variable_manager._extra_vars = {"ansible_ssh_pass": self.ssh_pass}
        print(self.variable_manager.get_vars())

    def run_model(self, module_name, module_args):
        """
        run module from andible ad-hoc.
        module_name: ansible module_name
        module_args: ansible module args
        """

        play_source = dict(
            name="Ansible Play",
            hosts=self.hosts,
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
                passwords=None,
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

    def run_playbook(self, playbook_path, extra_vars=None):
        """
        运行playbook
        """
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
            executor._tqm._stdout_callback = self.callback
            constants.HOST_KEY_CHECKING = False
            executor.run()
        except Exception as err:
            print(traceback.print_exc())
            return False

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
        self.results_raw = {'skipped': {}, 'failed': {}, 'ok': {}, 'unreachable': {}, 'status': {}}
        for host, result in self.callback.task_ok.items():
            self.results_raw['ok'][host] = result._result
        self.results_raw['ok']['number'] = len(self.callback.task_ok.items())

        for host, result in self.callback.task_failed.items():
            self.results_raw['failed'][host] = result._result
        self.results_raw['failed']['number'] = len(self.callback.task_failed.items())

        for host, result in self.callback.task_skipped.items():
            self.results_raw['skipped'][host] = result._result
        self.results_raw['skipped']['number'] = len(self.callback.task_skipped.items())

        for host, result in self.callback.task_unreachable.items():
            self.results_raw['unreachable'][host] = result._result
        self.results_raw['unreachable']['number'] = len(self.callback.task_unreachable.items())

        for host, result in self.callback.task_status.items():
            self.results_raw['status'][host] = result

        return self.results_raw


if __name__ == '__main__':
    rbt = AnsibleRunner(module_path='/Users/minsu/Documents/minsu/ansible/ocp_ansible_module', inventory='/Users/minsu/Documents/minsu/ansible/ocp_ansible_module/environment/dev-ocp2/hosts_multihost')  rbt.run_playbook(playbook_path='/Users/minsu/Documents/minsu/ansible/ocp_ansible_module/roles/haproxy/tasks/main.yml', extra_vars={"ansible_ssh_pass": '***', "ansible_sudo_pass": "***"})
    result = json.dumps(rbt.get_playbook_result(),indent=4)
    print(result)
```

* 初始化的时候`config_file='/Users/minsu/Documents/minsu/ansible/ansible.cfg'`导入ansible配置文件

* `verbosity=4`必须是数字，否则会报错，后面会比较大小
* `syntax=None, start_at_task=None`，必须有定义，否则会报key不存在
* `self.inventory.subset(['ci81hf1cmp003','ci81hf1cmp001','ci81hf1cmp002'])`，就是`--limit`一样