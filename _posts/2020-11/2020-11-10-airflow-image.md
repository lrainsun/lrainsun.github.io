---
layout: post
title:  "airflow kubernetes executor指定image"
date:   2020-11-10 23:00:00 +0800
categories: Airflow
tags: Airflow-Tutorial
excerpt: airflow kubernetes executor指定image
mathjax: true
typora-root-url: ../

---

# airflow kubernetes executor指定image

有个问题，搞了四天都没搞定

我们airflow会用来支持ocp 4.0&2.0，跑ansible deploy，在airflow image中安装了2.10的ansible，用这个跑4.0是没问题的，但是我们2.0用的是2.4.2的ansible，在有些语法上会跟2.10不太一样，比如一些filter。所以我们考虑在跑ansible的这个task的时候指定image

根据[这里](https://airflow.readthedocs.io/en/latest/executor/kubernetes.html)，可以这样指定

```python
    t_reconcile_ocp_compute_node_4_servers = SanityCheckPythonOperator(
        task_id='reconcile_ocp_compute_node_4_servers', trigger_rule='all_done',
        provide_context=True,
        python_callable=reconcile_ocp_compute_node_4_servers,
        op_kwargs={'build_id': '{{ build.id }}'},
        executor_config={"KubernetesExecutor": {"image": "registry-qa.webex.com/ocp/airflow-ocp:v1.10.12.1-1"}},
        executor_config={
            "pod_override": k8s.V1Pod(
                spec=k8s.V1PodSpec(
                    containers=[
                        k8s.V1Container(
                            name="ansible-2.4.2",
                            image="registry-qa.webex.com/ocp/airflow-ocp-ansible-2.4.2:v1.10.12-1",
                        ),
                    ],
                )
            ),
        },
    )
```

或者

```python
    t_reconcile_ocp_compute_node_4_servers = SanityCheckPythonOperator(
        task_id='reconcile_ocp_compute_node_4_servers', trigger_rule='all_done',
        provide_context=True,
        python_callable=reconcile_ocp_compute_node_4_servers,
        op_kwargs={ \
            'build_id': '{{ build.id }}',  \
            'dep_upstream_tasks' :  \
                 { \
                     'reconcile_boot_order_4_server_': extract_sn_from_task_id \
                } \
        },
        executor_config={"KubernetesExecutor": {"image": "registry-qa.webex.com/ocp/airflow-ocp-ansible-2.4.2:v1.10.12-1"}},
    )

```

我调试的环境kubernetes是1.18.2版本，会报这个错：

```shell
File "/home/airflow/.local/lib/python3.7/site-packages/flask/json/init.py", line 290, in htmlsafe_dumps
dumps(obj, **kwargs)
File "/home/airflow/.local/lib/python3.7/site-packages/flask/json/init.py", line 211, in dumps
rv = _json.dumps(obj, **kwargs)
File "/home/airflow/.local/lib/python3.7/site-packages/simplejson/init.py", line 412, in dumps
**kw).encode(obj)
File "/home/airflow/.local/lib/python3.7/site-packages/simplejson/encoder.py", line 296, in encode
chunks = self.iterencode(o, _one_shot=True)
File "/home/airflow/.local/lib/python3.7/site-packages/simplejson/encoder.py", line 378, in iterencode
return _iterencode(o, 0)
File "/home/airflow/.local/lib/python3.7/site-packages/flask/json/init.py", line 100, in default
return _json.JSONEncoder.default(self, o)
File "/home/airflow/.local/lib/python3.7/site-packages/simplejson/encoder.py", line 273, in default
o.class.name)
TypeError: Object of type V1Pod is not JSON serializable
```

而在kubernetes 1.14版本就可以的，但是遇到了新问题，在实验dag里指定Image都是没问题的，但是在我们的dag里始终有问题，查了好久，最终发现，Import了这两个包，跑ansible的时候需要用到的，加了这两个Import，再指定Image，就会让pod run不起来，目前还不知道是什么原因。

```python
from ansible.executor.task_queue_manager import TaskQueueManager
from ansible.executor.playbook_executor import PlaybookExecutor
```

