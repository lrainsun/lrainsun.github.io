---
layout: post
title:  "docker-py版本不一致导致的错误"
date:   2020-12-02 23:00:00 +0800
categories: Airflow
tags: Airflow-Tutorial
excerpt: docker-py版本不一致导致的错误
mathjax: true
typora-root-url: ../
---

# docker-py版本不一致导致的错误

我们在通过airflow部署telemetry的时候，发现会报这样的错误，我们的telemetry是跑在container里的

```shell
[2020-11-20 14:39:32,566] {logging_mixin.py:112} WARNING - fatal: [ci02qacmp027]: FAILED! => {"changed": false, "details": "{\"changed\": false, \"deprecations\": [{\"collection_name\": \"community.general\", \"msg\": \"The container_default_behavior option will change its default value from \\\"compatibility\\\" to \\\"no_defaults\\\" in community.general 3.0.0. To remove this warning, please specify an explicit value for it now\", \"version\": \"3.0.0\"}], \"msg\": \"Error updating container d38566a7594ace51aaf54284d9bdaf40989698859d7bc328472caabfe780a6e1: update_container() got an unexpected keyword argument 'restart_policy'\", \"warnings\": [\"log_options is ignored when log_driver is not specified\"]}", "ignore_errors": null, "msg": "Error updating container d38566a7594ace51aaf54284d9bdaf40989698859d7bc328472caabfe780a6e1: update_container() got an unexpected keyword argument 'restart_policy'"}
```

字面意思就是在更新container的时候，找不到restart_policy这个参数。一开始不理解是什么意思，经过一番排查才知道跟docker的lib库有关系

我们的ansible跑在airflow里，用的是最新版本的ansible 2.10，而我们要起的容器在host上，docker-py版本是`python-docker-py-1.10.6-11.el7.noarch`

在ansible 2.10版本中，`home/airflow/.local/lib//python3.7/site-packages/ansible_collections/community/general/plugins/modules/docker_container.py`

![img](/../assets/images/4597778.png)

![img](/../assets/images/4647523.png)

可以看到是支持了restart_policy这个update parameters的，所以在update container的时候，ansible就会把这个参数传给host上的docker-py，而我们看看docker-py 1.10.6版本的代码

`/usr/lib//python3.7/site-packages/ansible/modules/cloud/docker/docker_container.py`

```python
    @property
    def update_parameters(self):
        '''
        Returns parameters used to update a container
        '''

        update_parameters = dict(
            blkio_weight='blkio_weight',
            cpu_period='cpu_period',
            cpu_quota='cpu_quota',
            cpu_shares='cpu_shares',
            cpuset_cpus='cpuset_cpus',
            cpuset_mems='cpuset_mems',
            mem_limit='memory',
            mem_reservation='memory_reservation',
            memswap_limit='memory_swap',
            kernel_memory='kernel_memory',
        )
```

可以看到我们host上确实是不支持这个参数的，这个参数要docker-py 2.0.0以上版本才支持

由于我们是一套airflow，需要同时支持旧版本跟新版本，所以在这里做了个workaround，修改了一下ansible里的代码

```python
    def container_update(self, container_id, update_parameters):
        if update_parameters:
            self.log("update container %s" % (container_id))
            self.log(update_parameters, pretty_print=True)
            self.results['actions'].append(dict(updated=container_id, update_parameters=update_parameters))
            self.results['changed'] = True
            if not self.check_mode and callable(getattr(self.client, 'update_container')):
                try:
                    result = self.client.update_container(container_id, **update_parameters)
                    self.client.report_warnings(result)
                except Exception as exc:
                    try:
                        update_parameters.pop('restart_policy')
                        result = self.client.update_container(container_id, **update_parameters)
                    except Exception as exc:
                        self.fail("Error updating container %s: %s" % (container_id, to_native(exc)))
        return self._get_container(container_id)
```

先尝试update_container，如果失败，那么再去掉restart_policy参数再试一下

