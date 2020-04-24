---
layout: post
title:  "Airflow动态生成task"
date:   2020-04-23 23:00:00 +0800
categories: Airflow
tags: Airflow-Tutorial
excerpt: Airflow动态生成task
mathjax: true
typora-root-url: ../
---

# Airflow动态生成task

一个纠结了一天多的问题：如果我想动态生成一些airflow task要咋办？

比如我现在要去获取一个node的list，在我没获取到之前，我也不知道有几个node，等我获取到了，我要动态的根据node的多少，为每一个node就去建立一个task（这样就可以并行地为每个node都去做一些事情）

就这个问题，搞了我一天多都没搞定。。

References [[1]([https://stackoverflow.com/questions/41517798/proper-way-to-create-dynamic-workflows-in-airflow](https://stackoverflow.com/questions/41517798/proper-way-to-create-dynamic-workflows-in-airflow))] 里有那么多跟我一样纠结的人儿，可似乎都没找到好的方法

看起来airflow在生成task的时候，并不是在运行态生成的，而是需要提前算好的，比如写死，比如从全局变量里读，比如从预定义的文件里读，等等。但是，要是想根据前一个task的output来动态生成，这个事情好像是做不到

> If you are creating tasks dynamically you must do so **by iterating over something which is not created by an upstream task, or can be defined independently of that task**

然后就好纠结啊，咋办呢，目前能想到一个办法，就是比较傻

```python
def node_exist(number, **context):
    nodes = context['task_instance'].xcom_pull(task_ids='reconcile_db_records')
    if len(nodes) <= number:
        return False
    return True

def node_cimc_ip_and_login(number, **context):
    nodes = context['task_instance'].xcom_pull(task_ids='reconcile_db_records')
    ocp_deployment = context['task_instance'].xcom_pull(task_ids='fetch_infos_fetch_ocp_config')
    node = nodes[number]
    result, message = cimc.reconcile_node_cimc_ip_and_login(ocp_deployment, node)
    print(message)
    return result

for i in range(int(Variable.get('MAX_NODE'))):
    check_node = ShortCircuitOperator(
            task_id='check_node_' + str(i + 1),
            provide_context=True,
            python_callable=node_exist,
            op_kwargs={'number': i},
            dag=dag,
    )

    reconcile_node_cimc_ip_and_login = ShortCircuitOperator(
            task_id='reconcile_node_cimc_ip_and_login_' + str(i + 1),
            provide_context=True,
            python_callable=node_cimc_ip_and_login,
            op_kwargs={'number': i},
            dag=dag,
    )
    reconcile_db_records >> check_node >> reconcile_node_cimc_ip_and_login
```

需要一个最大值的配置，生成MAX_NODE个task，然后根据这个配置，去检查node是否真的存在，配合ShortCircuitOperator来达到skip的目的，效果是这样的。虽然我们只有不到10个node，可还是会创建10个task。目前为止似乎没找到更好的办法。

![image-20200423225759994](/../assets/images/image-20200423225759994.png)

![image-20200424154637554](/../assets/images/image-20200424154637554.png)

# References

[1] [https://stackoverflow.com/questions/41517798/proper-way-to-create-dynamic-workflows-in-airflow](https://stackoverflow.com/questions/41517798/proper-way-to-create-dynamic-workflows-in-airflow)