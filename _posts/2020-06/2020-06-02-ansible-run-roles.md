---
layout: post
title:  "Ansible Python API run roles"
date:   2020-06-02 23:00:00 +0800
categories: Ansible
tags: Ansible-pythonapi
excerpt: Ansible Python API run roles
mathjax: true
typora-root-url: ../
---

# Ansible Python API run roles

如果想在airflow里运行ansible，最好的还是能够把ansible playbook拆分开来，这样才能发挥airflow的优势，一旦失败，只需要re-run failed的部分就好了

加了一个run_roles的函数，需要在ansible.cfg里配置roles_path，绝对路径或者相对路径都可以，否则会找不到roles

```python
from ansible import constants
from collections import namedtuple
from ansible import context
from ansible.module_utils.common.collections import ImmutableDict
from ansible.parsing.dataloader import DataLoader
from ansible.playbook.play import Play
from ansible.s.task_queue_manager import TaskQueueManager
from ansible.executor.playbook_executor import PlaybookExecutor
from ansible.plugins.callback import CallbackBase
from ansible.plugins.callback.default import CallbackModule
from ansible.inventory.manager import InventoryManager
from ansible.vars.manager import VariableManager
import json


class PlayBookResultsCollector(CallbackModule):
    CALLBACK_VERSION = 2.0

    def __init__(self, *args, **kwargs):
        super(PlayBookResultsCollector, self).__init__(*args, **kwargs)
        self.task_ok = {}
        self.task_skipped = {}
        self.task_failed = {}
        self.task_status = {}
        self.task_unreachable = {}
        self.display_skipped_hosts = True
        self.display_ok_hosts = True
        self.show_custom_stats = True
        self.display_failed_stderr = True
        self.check_mode_markers = True

    def v2_runner_on_ok(self, result, *args, **kwargs):
        self.task_ok[result._host.get_name()] = result
        super().v2_runner_on_ok(result, *args, **kwargs)
        #print(result.__dict__)

    def v2_runner_on_failed(self, result, *args, **kwargs):
        self.task_failed[result._host.get_name()] = result
        super().v2_runner_on_failed(result, *args, **kwargs)

    def v2_runner_on_unreachable(self, result):
        self.task_unreachable[result._host.get_name()] = result
        super().v2_runner_on_unreachable(result)

    def v2_runner_on_skipped(self, result):
        self.task_ok[result._host.get_name()] = result
        super().v2_runner_on_skipped(result)

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

    def __init__(self, hosts=None, module_path='', roles_path='', inventory='', ssh_pass='', *args, **kwargs):
        self.hosts = hosts
        self.inventory = None
        self.variable_manager = None
        self.loader = None
        self.inventory = inventory
        self.ssh_pass = ssh_pass
        self.callback = None
        self.module_path = module_path
        self.roles_path = roles_path
        self.__initializeData()
        self.results_raw = {}

    def __initializeData(self):
        context.CLIARGS = ImmutableDict(connection='smart', module_path=[self.module_path], roles_path=self.roles_path, forks=10, become=True, become_method='sudo', become_user='root', become_ask_pass=False, check=False, diff=False, verbosity=4, syntax=None, start_at_task=None, config_file='ansible.cfg')

        self.loader = DataLoader()
        self.inventory = InventoryManager(loader=self.loader, sources=self.inventory)
        self.variable_manager = VariableManager(loader=self.loader, inventory=self.inventory)
        self.variable_manager._extra_vars = {"ansible_ssh_pass": self.ssh_pass}
        print(self.variable_manager.get_vars())

    def run_roles(self, hosts, roles, extra_vars):
        play_source = dict(
            name="Ansible Play",
            hosts=hosts,
            gather_facts='no',
            roles=roles
        )

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
        except Exception as err:
            print(traceback.print_exc())
        finally:
            if tqm is not None:
                tqm.cleanup()

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
    rbt = AnsibleRunner(inventory='/Users/minsu/Documents/minsu/ansible/ocp_ansible_module/environment/dev-ocp2/hosts_multihost')
    rbt.run_roles('controller', ['haproxy','keepalived'], extra_vars={"ansible_ssh_pass": '****', "ansible_sudo_pass": ""})

    result = json.dumps(rbt.get_playbook_result(),indent=4)
    print(result)
```

值得注意的是，roles_path的定义

可以在dataloader里set_basedir

```shell
        self.loader = DataLoader()
        self.loader.set_basedir('/opt/airflow/dags/ocp_ansible_module')
```

也可以配置在config_file里

```python
context.CLIARGS = ImmutableDict(connection='smart', module_path=[self.module_path], roles_path=self.roles_path, forks=10, become=True, become_method='sudo', become_user='root', become_ask_pass=False, check=False, diff=False, verbosity=4, syntax=None, start_at_task=None, config_file='ansible.cfg')
```

在ansible.cfg里有默认的roles_path定义

```shelll
roles_path = ocp_ansible_module/roles
```

inventory跟roles_path定义都是可以相对路径的