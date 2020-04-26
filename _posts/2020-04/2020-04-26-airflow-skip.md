---
layout: post
title:  "Airflow中skip task"
date:   2020-04-26 23:00:00 +0800
categories: Airflow
tags: Airflow-Tutorial
excerpt: Airflow中skip task
mathjax: true
typora-root-url: ../
---

# Airflow中skip task

前面试过dynamic task，只能目前用固定Number的方式来做，那么意味着，我们后面需要有一些task被skip掉。最开始用了ShortCircuirOperator，当return值为False的时候，downstream的task就都被被skip掉

![image-20200426204052238](/../assets/images/image-20200426204052238.png)

可以看到后面的reconcile_nodes_ips_and_hostnames也被skip掉了。这不是我们期望的。后来查到可以添加TriggerRule

> **trigger_rule** ([*str*](https://docs.python.org/3/library/stdtypes.html#str)) – defines the rule by which dependencies are applied for the task to get triggered. Options are: `{ all_success | all_failed | all_done | one_success | one_failed | none_failed | none_failed_or_skipped | none_skipped | dummy}` default is `all_success`. Options can be set as string or using the constants defined in the static class `airflow.utils.TriggerRule`

可是不管是加`none_failed_or_skipped`还是`none_failed`，reconcile_nodes_ips_and_hostnames都还是被skip掉了，这个行为并不是TriggerRule决定的，而是因为ShortCircuirOperator的作用，所有downstream的task都被skip了。这个方法行不通

后来想了办法，直接在reconcile_node_not_in_use里把异常处理掉，不让他失败也不让他skip，然后后面的reconcile_nodes_ips_and_hostnames就可以执行了，像这样

![image-20200426204616893](/../assets/images/image-20200426204616893.png)

觉得总还是不太好，于是又想了个办法，python operator里还有一个BranchPythonOperator，BranchPythonOperator的返回值，是一个taskid或者一个taskid的list，就是会被执行的next task，也就是比如我task A后面跟着有taskb1, taskb2, taskb3，然后taska是个BranchPythonOperator，并且返回taskb1，那么taskb2跟taskb3都会被skip掉。所以最后改成这样的效果，还是比较符合预期的

![image-20200426205445672](/../assets/images/image-20200426205445672.png)

