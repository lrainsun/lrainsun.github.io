---
layout: post
title:  "airflow为task pod添加init container"
date:   2020-11-26 23:00:00 +0800
categories: Airflow
tags: Airflow-Tutorial
excerpt: airflow为task pod添加init container
mathjax: true
typora-root-url: ../
---

# airflow为task pod添加init container

我们service需要跟vault集成来获取相关的一些密码等，由于airflow跑在kubernetes上，需要跑一个init container用来获取vault token。task的pod是由airflow schduler起起来的，那么怎么给task pod添加额外的Init container呢？airflow官方是这么说的：

## Pod Mutation Hook

The Airflow local settings file (`airflow_local_settings.py`) can define a `pod_mutation_hook` function that has the ability to mutate pod objects before sending them to the Kubernetes client for scheduling. It receives a single argument as a reference to pod objects, and is expected to alter its attributes.

This could be used, for instance, to add sidecar or init containers to every worker pod launched by KubernetesExecutor or KubernetesPodOperator.

```python
def pod_mutation_hook(pod: Pod):
  pod.annotations['airflow.apache.org/launched-by'] = 'Tests'
```

有这样一个配置，airflow_local_settings.py，放在`$AIRFLOW_HOME/config/airflow_local_settings.py`，在这里定义一个函数`pod_mutation_hook`，这个函数里可以在task pod启动之前，去修改pod的配置。

# 实践

所以我们可以这样，为了把这个配置文件放到config目录，我们首先定义一个configmap，内容就是我们想要添加的init container的内容

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: airflow-config
  namespace: iaas-airflow
data:
  airflow_local_settings.py: |
    from airflow.contrib.kubernetes.pod import Pod
    from airflow.kubernetes.volume import Volume
    import kubernetes.client.models as k8s

    def pod_mutation_hook(pod: Pod):
        extra_labels = {
            "test-label": "True",
        }
        pod.labels.update(extra_labels)

        print('pod before %s' % pod.__dict__)
        init_container = k8s.V1Container(env=[{'name':'VAULT_ROLE_ID', 'value':'********'}], image='registry-qa.webex.com/oke/kubernetes-vault-init:0.0.1', image_pull_policy='IfNotPresent', name='get-vault-token', termination_message_path='/dev/termination-log', termination_message_policy='File', volume_mounts=[{'mountPath':'/var/run/secrets/boostport.com', 'name':'vault-token'}])

        pod.init_containers = [init_container]

        volume = Volume(name='vault-token', configs={'emptyDir': {}})
        pod.volumes = pod.volumes + [volume]

        pod.annotations.update({
            'pod.boostport.com/vault-approle': 'iaas-airflow',
            'pod.boostport.com/vault-init-container': 'get-vault-token',
        })

        pod.volume_mounts = [{'mountPath':'/var/run/secrets/boostport.com', 'name':'vault-token'}]
        print('pod after %s' % pod.__dict__)
```

然后给pod添加extra mountpoint

```yaml
airflow:
  extraConfigmapMounts:
  - configMap: airflow-config
    mountPath: /opt/airflow/config/airflow_local_settings.py
    name: airflow-config
    readOnly: true
    subPath: airflow_local_settings.py
```

这样会把刚刚的configmap mount到指定的位置

除此之外，我们还需要需改airflow.cfg的一个配置`airflow_local_settings_configmap`

![image-20201126152137923](/../assets/images/image-20201126152137923.png)

我们可以通过修改环境变量来改变这个配置

```yaml
AIRFLOW__KUBERNETES__AIRFLOW_LOCAL_SETTINGS_CONFIGMAP: airflow-config
```

airflow配置是这样一级一级被覆盖的

```shell
The universal order of precedence for all configuration options is as follows:

set as an environment variable (AIRFLOW__CORE__SQL_ALCHEMY_CONN)

set as a command environment variable (AIRFLOW__CORE__SQL_ALCHEMY_CONN_CMD)

set as a secret environment variable (AIRFLOW__CORE__SQL_ALCHEMY_CONN_SECRET)

set in airflow.cfg

command in airflow.cfg

secret key in airflow.cfg

Airflow’s built in defaults
```

