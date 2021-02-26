---
layout: post
title:  "Ansible Python API run model"
date:   2020-11-06 23:00:00 +0800
categories: Ansible
tags: Ansible-pythonapi
excerpt: Ansible Python API run model
mathjax: true
typora-root-url: ../
---

# Ansible Python API run model

```python
class ModelResultsCollector(CallbackModule):
    def __init__(self, *args, **kwargs):
        super(ModelResultsCollector, self).__init__(*args, **kwargs)
        self.host_ok = {}
        self.host_unreachable = {}
        self.host_failed = {}
        self.display_skipped_hosts = True
        self.display_ok_hosts = True
        self.display_failed_stderr = True
        self.show_per_host_start = True

    def v2_runner_on_unreachable(self, result, *args, **kwargs):
        self.host_unreachable[result._host.get_name()] = result
        super().v2_runner_on_unreachable(result, *args, **kwargs)

    def v2_runner_on_ok(self, result, *args, **kwargs):
        self.host_ok[result._host.get_name()] = result
        super().v2_runner_on_ok(result, *args, **kwargs)

    def v2_runner_on_failed(self, result, *args, **kwargs):
        self.host_failed[result._host.get_name()] = result
        super().v2_runner_on_failed(result, *args, **kwargs)

class AnsibleRunner(object):
    """
    This is a General object for parallel execute modules.
    """

    def __init__(self, hosts=None, module_path='', roles_path='', inventory='', base_url='', ssh_pass='', private_key_file='', extra_vars={}, tags=[], skip_tags=[], *args, **kwargs):
        self.hosts = hosts
        self.inventory = None
        self.variable_manager = None
        self.loader = None
        self.inventory = inventory
        self.base_url = base_url
        self.ssh_pass = ssh_pass
        self.callback = None
        self.module_path = module_path
        self.roles_path = roles_path
        self.private_key_file = private_key_file
        self.extra_vars = extra_vars
        self.tags = tags
        self.skip_tags = skip_tags
        self.bastion_server = bastion_server
        self.__initializeData()
        self.results_raw = {}

    def __initializeData(self):
        context.CLIARGS = ImmutableDict(connection='smart', module_path=[self.module_path], roles_path=self.roles_path, forks=50, private_key_file=self.private_key_file, become=True, become_method ='sudo', become_user='root', become_ask_pass=False, check=False, diff=False, verbosity=4, syntax=None, start_at_task=None, extra_vars=self.extra_vars, config_file='/opt/airflow/dags/ansible.cfg', tags=self.tags, skip_tags=self.skip_tags)

        self.loader = DataLoader()
        self.loader.set_basedir(self.base_url)
        if type(self.inventory) != InventoryManager:
            self.inventory = InventoryManager(loader=self.loader, sources=self.inventory)
        self.variable_manager = VariableManager(loader=self.loader, inventory=self.inventory)
        self.hosts = list(self.inventory.hosts.keys())


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
            play = Play().load(play_source, variable_manager=self.variable_manager, loader=self.loader)
            tqm.run(play)
        except Exception as err:
            print(traceback.print_exc())
        finally:
            if tqm is not None:
                tqm.cleanup()

def _run_ansible_model(module_name, module_args, extra_vars = None, hosts = None):
    ansible_runner = AnsibleRunner(inventory = AIRFLOW_INVENTORY_PATH)
    if hosts:
        ansible_runner.inventory.subset(hosts)
    ansible_runner.run_model(module_name, module_args, extra_vars)
    result, tasks = ansible_runner.get_model_result()
    return result

def generate_ssh_dir(password, hosts):
    print('genereate ssh dir')
    return _run_ansible_model('file', {'name': '/home/' + OCPADMIN + '/.ssh', 'state': 'directory', 'owner': OCPADMIN, 'group': OCPADMIN}, extra_vars = {'ansible_ssh_pass': password}, hosts = hosts)
```

当没有现成的playbook，而执行的module又相对简单的时候，可以直接run module