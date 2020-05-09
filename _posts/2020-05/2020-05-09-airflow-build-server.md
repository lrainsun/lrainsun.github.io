---
layout: post
title:  "Airflow Fail提醒"
date:   2020-05-09 23:00:00 +0800
categories: Airflow
tags: Airflow-Tutorial
excerpt: Airflow Fail提醒
mathjax: true
typora-root-url: ../
---

# Fail提醒

这两天又跟airflow混在一起了

发现dag是否成功，是跟最后一个task是否成功有关系的，本来我在最后加了send email/teams message的operator，可以发送结果，但是如果中间的task失败了，除了本身airflow支持的email之外，我们还希望可以看到teams message。DAG的定义中有一个

> **on_failure_callback** (*callable*) – a function to be called when a task instance of this task fails. a context dictionary is passed as a single parameter to this function. Context contains references to related objects to the task instance and is documented under the macros section of the API.

可以在一个task失败的时候，调用这里的callback，那么我们就可以在这个callback里发送消息给teams message

```python
def failure_callback(context):
    """
    The function that will be executed on failure.

    :param context: The context of the executed task.
    :type context: dict
    """
    message = """## OCP Server Build Failed\n 
- - - \n
- **DAG**:    {}\n 
- **TASKS**:  {}\n 
- **Reason**: {}\n
- **Log**: [Log]({})""".format(context['task_instance'].dag_id,
                context['task_instance'].task_id,
                context['exception'],
                context['task_instance'].log_url)
    dag_id = context['task_instance'].dag_id
    task_id = context['task_instance'].task_id
    execption = context['exception']
    return TeamsOperator(
        task_id='failure_callback',
        access_token=access_token,
        spaces=spaces,
        markdown_content=message,
        dag=dag,
    ).execute(context)

default_args['on_failure_callback'] = failure_callback
```

