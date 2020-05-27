---
layout: post
title:  "Airflow中的contxt查询"
date:   2020-05-27 23:00:00 +0800
categories: Airflow
tags: Airflow-Tutorial
excerpt: Airflow中的contxt查询
mathjax: true
typora-root-url: ../
---

# Airflow中的contxt查询

经常会用到airflow中的context，不管是python callable function还是template，总忘但是，把用到的列一下

|                | python callable                                              | template                                                   |
| -------------- | ------------------------------------------------------------ | ---------------------------------------------------------- |
| conf           | context.get('dag_run').conf.get('req_num')                   | {{ dag_run.conf['req_num'] }}                              |
| execution date | context.get('execution_date')                                | {{ execution_date }}                                       |
| dag id         | context['task_instance'].dag_id                              | {{ ti.dag_id }}                                            |
| task id        | context['task_instance'].task_id                             | {{ ti.task_id }}                                           |
| exception      | context['exception']                                         | {{ exception }}                                            |
| xcom -- return | context['task_instance'].xcom_pull(task_ids='fetch_ocp_config') | {{ task_instance.xcom_pull(task_ids='fetch_ocp_config') }} |
| xcom -- key    | context['task_instance'].xcom_pull(key='teams_out_put')      | {{ task_instance.xcom_pull(key='teams_out_put') }}         |