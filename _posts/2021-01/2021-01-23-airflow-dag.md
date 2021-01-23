---
layout: post
title:  "airflow dag"
date:   2021-01-23 23:01:00 +0800
categories: Airflow
tags: Airflow-Tutorial
excerpt: airflow dag
mathjax: true
typora-root-url: ../
---

# airflow dag

前几天发生一个问题，我们写了个dag，因为这个dag是通过generate代码生成的，然后dag死活都加载不出来，后来看了代码，才发现之前的理解是错的。之前一直是以为只要放在/opt/airflow/dags下的py文件，文件名中包含"dag" or "airflow"，就会被认为是dag来加载，现在看来并不是这么一回事。

首先，我们可以去/opt/airflow/logs下查看dag load的日志：dag_processor_manager.log

```shell
DAG File Processing Stats

File Path                                                                                                                                                  PID  Runtime      # DAGs    # Errors  Last Runtime    Last Run
-------------------------------------------------------------------------------------------------------------------------------------------------------  -----  ---------  --------  ----------  --------------  -------------------
/opt/airflow/dags/0_common_dag.py                                                                                                                                                 0           0  3.36s           2021-01-18T15:50:54
/opt/airflow/dags/2_openstack_dag.py                                                                                                                                              0           0  3.02s           2021-01-18T15:51:27
/opt/airflow/dags/3_add_dag.py                                                                                                                                                    0           0  3.02s           2021-01-18T15:51:32
/opt/airflow/dags/airflow_dag_generator.py                                                                                                                                        0           0  1.01s           2021-01-18T15:51:37
/opt/airflow/dags/copy_sshkey_dag.py                                                                                                                                              0           0  3.02s           2021-01-18T15:51:55
/opt/airflow/dags/generate_inventory.py                                                                                                                   9193  0.00s             0           0  3.29s           2021-01-18T15:51:19
/opt/airflow/dags/kolla_ansible_allinone_dag.py                                                                                                                                   0           0  3.03s           2021-01-18T15:51:48
/opt/airflow/dags/kolla_ansible_dag.py                                                                                                                                            0           0  3.01s           2021-01-18T15:52:00
/opt/airflow/dags/kubernetes_executor_test.py                                                                                                                                     0           0  1.40s           2021-01-18T15:51:51
/opt/airflow/dags/ocp_ansible_module_dag.py   

[2021-01-18 15:55:20,923] {dag_processing.py:939} INFO - Searching for files in /opt/airflow/dags
[2021-01-18 15:55:21,250] {dag_processing.py:942} INFO - There are 46 files in /opt/airflow/dags
[2021-01-18 15:55:21,270] {dag_processing.py:1312} INFO - Finding 'running' jobs without a recent heartbeat
[2021-01-18 15:55:21,270] {dag_processing.py:1316} INFO - Failing jobs without heartbeat after 2021-01-18 15:50:21.270608+00:00
[2021-01-18 15:55:26,317] {dag_processing.py:939} INFO - Searching for files in /opt/airflow/dags
[2021-01-18 15:55:26,681] {dag_processing.py:942} INFO - There are 46 files in /opt/airflow/dags
[2021-01-18 15:55:31,728] {dag_processing.py:939} INFO - Searching for files in /opt/airflow/dags
[2021-01-18 15:55:32,008] {dag_processing.py:942} INFO - There are 46 files in /opt/airflow/dags
[2021-01-18 15:55:32,021] {dag_processing.py:1312} INFO - Finding 'running' jobs without a recent heartbeat
[2021-01-18 15:55:32,021] {dag_processing.py:1316} INFO - Failing jobs without heartbeat after 2021-01-18 15:50:32.021543+00:00
[2021-01-18 15:55:37,067] {dag_processing.py:939} INFO - Searching for files in /opt/airflow/dags
```

这边会把所有dag都load进来，连这种名字里面不带dag跟airflow都load进来了/opt/airflow/dags/generate_inventory.py 

那load规则到底是什么呢？

看了代码才恍然大悟

![image-20210123213034486](/../assets/images/image-20210123213034486.png)

![image-20210123213059328](/../assets/images/image-20210123213059328.png)

![image-20210123213123694](/../assets/images/image-20210123213123694.png)

原来，并不是文件名的问题，而是文件内容里面需要包含单独的“DAG”或者“airflow"才会被认为是dag来扫描

