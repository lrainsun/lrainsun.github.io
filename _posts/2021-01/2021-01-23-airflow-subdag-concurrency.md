---
layout: post
title:  "airflow subdag concurrency"
date:   2021-01-23 23:02:00 +0800
categories: Airflow
tags: Airflow-Tutorial
excerpt: airflow subdag concurrency
mathjax: true
typora-root-url: ../
---

# airflow subdag concurrency

我们在有些dag里面用了subdag，但是发现在subdag里面不能并行运行，总是串行的

https://stackoverflow.com/questions/56051276/airflow-1-10-3-subdag-can-only-run-1-task-in-parallel-even-the-concurrency-is-8

原来这跟版本有关系

**Airflow 1.9.0**: https://github.com/apache/airflow/blob/1.9.0/airflow/operators/subdag_operator.py#L33

```python
class SubDagOperator(BaseOperator):

    template_fields = tuple()
    ui_color = '#555'
    ui_fgcolor = '#fff'

    @provide_session
    @apply_defaults
    def __init__(
            self,
            subdag,
            executor=GetDefaultExecutor(),
            *args, **kwargs):
```

而在 Airflow 1.10之后，default executor for SubDagOperator改成了SequentialExecutor

**Airflow >=1.10**: https://github.com/apache/airflow/blob/1.10.0/airflow/operators/subdag_operator.py#L38

```python
class SubDagOperator(BaseOperator):

    template_fields = tuple()
    ui_color = '#555'
    ui_fgcolor = '#fff'

    @provide_session
    @apply_defaults
    def __init__(
            self,
            subdag,
            executor=SequentialExecutor(),
            *args, **kwargs):
```

workaround是创建subdag的时候指定executor

```python
from airflow.operators.subdag_operator import SubDagOperator
from airflow.executors import GetDefaultExecutor

def sub_dag_operator_with_default_executor(subdag, *args, **kwargs):
    return SubDagOperator(subdag=subdag, executor=GetDefaultExecutor(), *args, **kwargs)
```

