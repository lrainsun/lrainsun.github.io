---
layout: post
title:  "Airflow Executor"
date:   2020-04-20 23:00:00 +0800
categories: Airflow
tags: Airflow-Start
excerpt: Airflow Executor
mathjax: true
typora-root-url: ../
---

# Executor

Executor是task实例运行的机制。Airflow支持几种executor，在`~/airflow/airflow.cfg`中定义

SequentialExecutor, LocalExecutor, CeleryExecutor, DaskExecutor, KubernetesExecutor

## SequentialExecutor

默认是使用的SequentialExecutor，只能顺序执行task

SequentialExecutor可以配合sqlite数据库使用（*sqlite*简单好用，但是不支持多个进程的*并行*操作，即使*并行*的读也不行），方便快速上手

# LocalExecutor

LocalExecutor顾名思义就是本地的executor，需要安装mysql。

也就是所有的task都是跑在本地的，可以支持并发，一般做测试用。

# CeleryExecutor

可用于生产环境，可以支持多workers，也就是在不同的workers来运行task。CeleryExecutor需要backend的支持（RabbitMQ, Redis, ....）

# KubernetesExecutor

kubernetesexecutor会为每个task实例创建一个新的pod

# 并行

有几个参数可以用来控制airflow运行的并行度

## parallelism

```yaml
# The amount of parallelism as a setting to the executor. This defines
# the max number of task instances that should run simultaneously
# on this airflow installation
parallelism = 32
```

控制每个airflow worker 可以同时运行多少个task实例，这是airflow集群的全局变量，在airflow.cfg里面配置

## concurrency

```yaml
# The number of task instances allowed to run concurrently by the scheduler
dag_concurrency = 16
```

用来控制每个dag运行过程中最大可同时运行的task实例数，如果没有设置这个值，scheduler 会从airflow.cfg里面读取默认值 dag_concurrency

## max_active_runs

用来控制在同一时间可以运行的最多的dag runs 数量。比如你的dag设置的每天运行，那么在天的时间段内运行某个dag就算是一个dag runs 。按道理每天只会执行一次，但是保不齐，你前天和大前天的dag都没运行，那么就需要补跑，或者你在某一次定时dag触发了之后，又手动触发了，那么就存在，同一个时间点有多个dag runs 。这个参数就是控制这个最大的dag runs

```yaml
# The maximum number of active DAG runs per DAG
max_active_runs_per_dag = 16
```

# Pool

Pool可以用来限制同一个Pool下的task并发数，可以在UI上`Menu -> Admin -> Pools`定义：

在创建tasks（比如实例化Operators的时候）可以定义使用哪个pool，如果不定义，那么默认是在`default_pool`

```python
aggregate_db_message_job = BashOperator(
    task_id='aggregate_db_message_job',
    execution_timeout=timedelta(hours=3),
    pool='ep_data_pipeline_db_msg_agg',
    bash_command=aggregate_db_message_job_cmd,
    dag=dag)
aggregate_db_message_job.set_upstream(wait_for_empty_queue)
```

![image-20200421101940330](/../assets/images/image-20200421101940330.png)

`Pool`定义可以联合`priority_weight`定义来定义queue中的优先级，用来决定哪个task先执行，默认的`priority_weight`是1。

# References

[1] [https://airflow.apache.org/docs/stable/executor/index.html](https://airflow.apache.org/docs/stable/executor/index.html)