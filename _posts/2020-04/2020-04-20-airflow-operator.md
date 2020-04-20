---
layout: post
title:  "Airflow Operator"
date:   2020-04-20 23:00:00 +0800
categories: Airflow
tags: Airflow-Start
excerpt: Airflow Operator
mathjax: true
typora-root-url: ../
---

# BashOperator（bash执行）

格式是这样的：

```python
run_this = BashOperator(
    task_id='run_after_loop',
    bash_command='echo 1',
    dag=dag,
)
```

可以用jinja模板来给bash command参数化

```python
also_run_this = BashOperator(
    task_id='also_run_this',
    bash_command='echo "run_id={{ run_id }} | dag_run={{ dag_run }}"',
    dag=dag,
)
```

然后有个地方要注意的是，如果在bash command直接去调用一个bash script的话，需要在最后多加一个space

```python
t2 = BashOperator(
    task_id='bash_example',

    # This fails with 'Jinja template not found' error
    # bash_command="/home/batcher/test.sh",

    # This works (has a space after)
    bash_command="/home/batcher/test.sh ",
    dag=dag)
```

# PythonOperator

```python
def print_context(ds, **kwargs):
    pprint(kwargs)
    print(ds)
    return 'Whatever you return gets printed in the logs'


run_this = PythonOperator(
    task_id='print_the_context',
    provide_context=True,
    python_callable=print_context,
    dag=dag,
)
```

还可以通过op_args和op_kwargs传递额外的参数给python函数

```python
def my_sleeping_function(random_base):
    """This is a function that will run within the DAG execution"""
    time.sleep(random_base)


# Generate 5 sleeping tasks, sleeping from 0.0 to 0.4 seconds respectively
for i in range(5):
    task = PythonOperator(
        task_id='sleep_for_' + str(i),
        python_callable=my_sleeping_function,
        op_kwargs={'random_base': float(i) / 10},
        dag=dag,
    )

    run_this >> task
```

