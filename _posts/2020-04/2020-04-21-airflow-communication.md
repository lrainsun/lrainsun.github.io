---
layout: post
title:  "Airflow中的连接和参数传递"
date:   2020-04-21 23:00:00 +0800
categories: Airflow
tags: Airflow-Start
excerpt: Airflow中的连接和参数传递，Connections, queues, messages
mathjax: true
typora-root-url: ../
---

# Connections

对于kubernetes executor和celery executor来说，task会被放到不同的server上去运行。所以不能存放任何本地的文件或者配置。

通过XCom可以在task直接传递small messages，如果是大量数据的话（推荐使用remote storage such as S3/HDFS）

所有类似这样的airflow的外部连接，可以通过Connections来存储。

`Menu -> Admin -> Connections`这里可以管理Connections。比如mysql之类的连接可以放在这边定义。

# Queues

当使用CeletryExecutor的时候，queue可以specified，相当于贴个标签

queue是一个BaseOperator的属性值，any task can be assigned to any queue。

默认的queue可以在airflow.cfg里定义

```shell
# Default queue that tasks get assigned to and that worker listen on.
 525 default_queue = default
```

celetry workers起来的时候，可以监听一个或多个queue的task。在operator中，可以设置queue参数如queue=spark，比如在启动worker时：`airflow worker -q spark`，那么该worker只会执行spark任务。相当于节点标签。

# Conf

首先，在trigger一个DAG的时候，可以通过`-c`来传递一个配置

```shell
(airflow) MINSU-M-M1RW:~ minsu$ airflow trigger_dag -h
usage: airflow trigger_dag [-h] [-sd SUBDIR] [-r RUN_ID] [-c CONF]
                           [-e EXEC_DATE]
                           dag_id

positional arguments:
  dag_id                The id of the dag

optional arguments:
  -h, --help            show this help message and exit
  -sd SUBDIR, --subdir SUBDIR
                        File location or directory from which to look for the
                        dag. Defaults to '[AIRFLOW_HOME]/dags' where
                        [AIRFLOW_HOME] is the value you set for 'AIRFLOW_HOME'
                        config you set in 'airflow.cfg'
  -r RUN_ID, --run_id RUN_ID
                        Helps to identify this run
  -c CONF, --conf CONF  JSON string that gets pickled into the DagRun's conf
                        attribute
  -e EXEC_DATE, --exec_date EXEC_DATE
                        The execution date of the DAG
```

比如我们这样

```python
def print_context(ds, **kwargs):
     print(kwargs)
     print(ds)
     print(kwargs.get('dag_run').conf.get('name'))
     print(kwargs.get('dag_run').conf["name"])
     return 'Whatever you return gets printed in the logs'
```

通过命令行可以trigger一个dag并传入参数

```shell
(airflow) MINSU-M-M1RW:minsu minsu$ airflow trigger_dag test_python_dag -c '{"name":"rain"}'
[2020-04-20 16:46:52,624] {__init__.py:51} INFO - Using executor SequentialExecutor
[2020-04-20 16:46:52,625] {dagbag.py:396} INFO - Filling up the DagBag from /Users/minsu/airflow/dags/python.py
Created <DagRun test_python_dag @ 2020-04-20 08:46:52+00:00: manual__2020-04-20T08:46:52+00:00, externally triggered: True>
```

在UI上可以这样：

![image-20200420163354689](/../assets/images/image-20200420163354689.png)

查看打印，参数值被打印出来

![image-20200420165045545](/../assets/images/image-20200420165045545.png)

这里还打印了kwargs，可以看看包含哪些内容

```python
def print_context(ds, **kwargs):
    print(kwargs)
    print(ds)
    return 'Whatever you return gets printed in the logs'


run_this = PythonOperator(
    task_id='print_the_context',
    provide_context=True,
    python_callable=print_context,
    dag=dag,
)
```

`provide_context=True`是必要的，因为需要使用PythonOperator 的provide_context=True 来获取外界参数。 provide_context 默认为False，即不会将产生的数据放入上下文中。provide_context 设置为True后， 在自定义方法的参数中注入 **kwargs ，从**kwargs 中获取外界传来的数据。

```shell
[2020-04-20 16:49:29,873] {logging_mixin.py:112} INFO - {'conf': <airflow.configuration.AirflowConfigParser object at 0x1066015f8>, 'dag': <DAG: test_python_dag>, 'next_ds': '2020-04-20', 'next_ds_nodash': '20200420', 'prev_ds': '2020-04-20', 'prev_ds_nodash': '20200420', 'ds_nodash': '20200420', 'ts': '2020-04-20T08:47:48.140709+00:00', 'ts_nodash': '20200420T084748', 'ts_nodash_with_tz': '20200420T084748.140709+0000', 'yesterday_ds': '2020-04-19', 'yesterday_ds_nodash': '20200419', 'tomorrow_ds': '2020-04-21', 'tomorrow_ds_nodash': '20200421', 'END_DATE': '2020-04-20', 'end_date': '2020-04-20', 'dag_run': <DagRun test_python_dag @ 2020-04-20 08:47:48.140709+00:00: manual__2020-04-20T08:47:48.140709+00:00, externally triggered: True>, 'run_id': 'manual__2020-04-20T08:47:48.140709+00:00', 'execution_date': <Pendulum [2020-04-20T08:47:48.140709+00:00]>, 'prev_execution_date': <Pendulum [2020-04-20T08:47:48.140709+00:00]>, 'prev_execution_date_success': <Proxy at 0x113208c48 with factory <function TaskInstance.get_template_context.<locals>.<lambda> at 0x113182c80>>, 'prev_start_date_success': <Proxy at 0x11318ac88 with factory <function TaskInstance.get_template_context.<locals>.<lambda> at 0x11327a048>>, 'next_execution_date': <Pendulum [2020-04-20T08:47:48.140709+00:00]>, 'latest_date': '2020-04-20', 'macros': <module 'airflow.macros' from '/Users/minsu/.pyenv/versions/3.6.0/envs/airflow/lib/python3.6/site-packages/airflow/macros/__init__.py'>, 'params': {}, 'tables': None, 'task': <Task(PythonOperator): print_the_context>, 'task_instance': <TaskInstance: test_python_dag.print_the_context 2020-04-20T08:47:48.140709+00:00 [running]>, 'ti': <TaskInstance: test_python_dag.print_the_context 2020-04-20T08:47:48.140709+00:00 [running]>, 'task_instance_key_str': 'test_python_dag__print_the_context__20200420', 'test_mode': False, 'var': {'value': None, 'json': None}, 'inlets': [], 'outlets': [], 'templates_dict': None}
```

这边有两个，一个var，一个templates_dict，感觉都是可以传参的，可以研究下

When you set the `provide_context` argument to `True`, Airflow passes in an additional set of keyword arguments: one for each of the [Jinja template variables](https://airflow.apache.org/docs/stable/macros-ref.html) and a `templates_dict` argument.

The `templates_dict` argument is templated, so each value in the dictionary is evaluated as a [Jinja template](https://airflow.apache.org/docs/stable/concepts.html#jinja-templating).

# 内部传参

```python
templated_command = """
{% for i in range(5) %}
    echo "{{ ds }}"
    echo "{{ macros.ds_add(ds, 7)}}"
    echo "{{ params.my_param }}"
{% endfor %}
"""

t3 = BashOperator(
    task_id='templated',
    depends_on_past=False,
    bash_command=templated_command,
    params={'my_param': 'Parameter I passed in'},
    dag=dag,
)
```

可以通过这样的方式，把参数传递给callback

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
```

# XComs

XComs是`cross-communication`的缩写，默认情况下，dag与dag之间 、task与task之间信息是无法共享的。如果想在dag、task之间实现信息共享，可以使用XComs，通过设置在一个dag(task)中设置XComs参数在另一个中读取来实现信息共享。

通常XComs可以定义为一个key, value或者timestamp，也可以是track attributes ——> 任何可提取的对象都可以被用来作为XCom value

XComs可以被"push"（发送）或者"pulled"（接受）

## push

一个task push（`xcom_push()`）了一个XCom，那么对其他tasks就是可见的

如果一个task返回（return）了一个值（或者是Operator的execute()方法返回的，或者是PythonOperator的python_callable返回的），一个包含了返回值的XCom会自动被push

## pull

task可以调用`xcom_pull()`来获取XComs，可以通过filter来过滤，比如key，task_ids，source dag_id等。

`xcom_pull()`默认会filter keys（被execute push的时候自动加上的）

```python
# inside a PythonOperator called 'pushing_task'
def push_function():
    return value

# inside another PythonOperator where provide_context=True
def pull_function(**context):
    value = context['task_instance'].xcom_pull(task_ids='pushing_task')
```

### 例子1

return值自动push

```python
def print_context(ds, **kwargs):
    print(kwargs)
    print(ds)
    print(kwargs.get('dag_run').conf.get('name'))
    print(kwargs.get('dag_run').conf["name"])
    return 'get return value by xcoms'


run_this = PythonOperator(
    task_id='print_the_context',
    provide_context=True,
    python_callable=print_context,
    dag=dag,
)
#-------------------------------------------------------------------------------
def print_last_return_value(**context):
    value = context['task_instance'].xcom_pull(task_ids='print_the_context')
    print(value)
    return value

print_operator = PythonOperator(
        task_id='print_task',
        python_callable=print_last_return_value,
        provide_context=True,
        dag=dag,
)
```

跑完之后print_task的打印

![image-20200421134124961](/../assets/images/image-20200421134124961.png)

### 例子2

```python
def print_context(ds, **kwargs):
    print(kwargs)
    print(ds)
    print(kwargs.get('dag_run').conf.get('name'))
    print(kwargs.get('dag_run').conf["name"])
    task_instance = kwargs['task_instance']
    task_instance.xcom_push(key="test_xcom", value="can you get the value?")
    return 'get return value by xcoms'


run_this = PythonOperator(
    task_id='print_the_context',
    provide_context=True,
    python_callable=print_context,
    dag=dag,
)
#-------------------------------------------------------------------------------
def print_last_return_value(**context):
    value = context['task_instance'].xcom_pull(task_ids='print_the_context')
    print(value)
    key_value = context['task_instance'].xcom_pull(task_ids='print_the_context',key='test_xcom')
    print(key_value)
    return value

print_operator = PythonOperator(
        task_id='print_task',
        python_callable=print_last_return_value,
        provide_context=True,
        dag=dag,
)

run_this >> print_operator
```

我们在run_this中push了一个key, value，在print_operator中就可以获取到

![image-20200421135209347](/../assets/images/image-20200421135209347.png)

# Variables

Variables是在Airflow中存储和检索任意内容或设置作为简单key-value存储的通用方法。

通过UI（Admin -> Variables）就可以list, create, updated, deleted，也可以upload json setting files。

![image-20200421150520291](/../assets/images/image-20200421150520291.png)

比如我在这里定义了一个global的variable，就可以在代码里这样

```python
from airflow.models import Variable

def print_context(ds, **kwargs):
    print(kwargs)
    print(ds)
    print(Variable.get("test_var"))
    print(kwargs.get('dag_run').conf.get('name'))
    print(kwargs.get('dag_run').conf["name"])
    task_instance = kwargs['task_instance']
    task_instance.xcom_push(key="test_xcom", value="can you get the value?")
    return 'get return value by xcoms'
```

![image-20200421150630136](/../assets/images/image-20200421150630136.png)

也可以在代码中设置Variable，其他task中读取

```python
def print_context(ds, **kwargs):
    print(kwargs)
    print(ds)
    print(Variable.get("test_var"))
    Variable.set("var_in_dag", "test vars in dag")
    print(kwargs.get('dag_run').conf.get('name'))
    print(kwargs.get('dag_run').conf["name"])
    task_instance = kwargs['task_instance']
    task_instance.xcom_push(key="test_xcom", value="can you get the value?")
    return 'get return value by xcoms'


run_this = PythonOperator(
    task_id='print_the_context',
    provide_context=True,
    python_callable=print_context,
    dag=dag,
)
#-------------------------------------------------------------------------------
def print_last_return_value(**context):
    value = context['task_instance'].xcom_pull(task_ids='print_the_context')
    print(value)
    key_value = context['task_instance'].xcom_pull(task_ids='print_the_context',key='test_xcom')
    print(key_value)
    variable = Variable.get("var_in_dag")
    print(variable)
    return value

print_operator = PythonOperator(
        task_id='print_task',
        python_callable=print_last_return_value,
        provide_context=True,
        dag=dag,
)
```

![image-20200421153437075](/../assets/images/image-20200421153437075.png)

The best way of using variables is via a Jinja template which will delay reading the value until the task execution. The template synaxt to do this is:

```
{{ var.value.<variable_name> }}
```

or if you need to deserialize a json object from the variable :

```
{{ var.json.<variable_name> }}
```

airflow预置了一些变量，可以直接用

The Airflow engine passes a few variables by default that are accessible in all templates

| Variable                            | Description                                                  |
| ----------------------------------- | ------------------------------------------------------------ |
| `{{ ds }}`                          | the execution date as `YYYY-MM-DD`                           |
| `{{ ds_nodash }}`                   | the execution date as `YYYYMMDD`                             |
| `{{ prev_ds }}`                     | the previous execution date as `YYYY-MM-DD` if `{{ ds }}` is `2018-01-08` and `schedule_interval` is `@weekly`, `{{ prev_ds }}` will be `2018-01-01` |
| `{{ prev_ds_nodash }}`              | the previous execution date as `YYYYMMDD` if exists, else `None` |
| `{{ next_ds }}`                     | the next execution date as `YYYY-MM-DD` if `{{ ds }}` is `2018-01-01` and `schedule_interval` is `@weekly`, `{{ next_ds }}` will be `2018-01-08` |
| `{{ next_ds_nodash }}`              | the next execution date as `YYYYMMDD` if exists, else `None` |
| `{{ yesterday_ds }}`                | the day before the execution date as `YYYY-MM-DD`            |
| `{{ yesterday_ds_nodash }}`         | the day before the execution date as `YYYYMMDD`              |
| `{{ tomorrow_ds }}`                 | the day after the execution date as `YYYY-MM-DD`             |
| `{{ tomorrow_ds_nodash }}`          | the day after the execution date as `YYYYMMDD`               |
| `{{ ts }}`                          | same as `execution_date.isoformat()`. Example: `2018-01-01T00:00:00+00:00` |
| `{{ ts_nodash }}`                   | same as `ts` without `-`, `:` and TimeZone info. Example: `20180101T000000` |
| `{{ ts_nodash_with_tz }}`           | same as `ts` without `-` and `:`. Example: `20180101T000000+0000` |
| `{{ execution_date }}`              | the execution_date (logical date) ([pendulum.Pendulum](https://pendulum.eustace.io/docs/1.x/#introduction)) |
| `{{ prev_execution_date }}`         | the previous execution date (if available) ([pendulum.Pendulum](https://pendulum.eustace.io/docs/1.x/#introduction)) |
| `{{ prev_execution_date_success }}` | execution date from prior successful dag run (if available) ([pendulum.Pendulum](https://pendulum.eustace.io/docs/1.x/#introduction)) |
| `{{ prev_start_date_success }}`     | start date from prior successful dag run (if available) ([pendulum.Pendulum](https://pendulum.eustace.io/docs/1.x/#introduction)) |
| `{{ next_execution_date }}`         | the next execution date ([pendulum.Pendulum](https://pendulum.eustace.io/docs/1.x/#introduction)) |
| `{{ dag }}`                         | the DAG object                                               |
| `{{ task }}`                        | the Task object                                              |
| `{{ macros }}`                      | a reference to the macros package, described below           |
| `{{ task_instance }}`               | the task_instance object                                     |
| `{{ end_date }}`                    | same as `{{ ds }}`                                           |
| `{{ latest_date }}`                 | same as `{{ ds }}`                                           |
| `{{ ti }}`                          | same as `{{ task_instance }}`                                |
| `{{ params }}`                      | a reference to the user-defined params dictionary which can be overridden by the dictionary passed through `trigger_dag -c` if you enabled `dag_run_conf_overrides_params` in ``airflow.cfg` |
| `{{ var.value.my_var }}`            | global defined variables represented as a dictionary         |
| `{{ var.json.my_var.path }}`        | global defined variables represented as a dictionary with deserialized JSON object, append the path to the key within the JSON object |
| `{{ task_instance_key_str }}`       | a unique, human-readable key to the task instance formatted `{dag_id}_{task_id}_{ds}` |
| `{{ conf }}`                        | the full configuration object located at `airflow.configuration.conf` which represents the content of your `airflow.cfg` |
| `{{ run_id }}`                      | the `run_id` of the current DAG run                          |
| `{{ dag_run }}`                     | a reference to the DagRun object                             |
| `{{ test_mode }}`                   | whether the task instance was called using the CLI’s test subcommand |

Note that you can access the object’s attributes and methods with simple dot notation. Here are some examples of what is possible: `{{ task.owner }}`, `{{ task.task_id }}`, `{{ ti.hostname }}`, … Refer to the models documentation for more information on the objects’ attributes and methods.

The `var` template variable allows you to access variables defined in Airflow’s UI. You can access them as either plain-text or JSON. If you use JSON, you are also able to walk nested structures, such as dictionaries like: `{{ var.json.my_dict_var.key1 }}`.

XComs跟Variables很相似，不同之处在于：

* XComs主要用来跨task的communication
* Variables主要是全局的配置

# References

[1] [https://airflow.apache.org/docs/stable/concepts.html#jinja-templating]( https://airflow.apache.org/docs/stable/concepts.html#jinja-templating)

