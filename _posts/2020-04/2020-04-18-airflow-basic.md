---
layout: post
title:  "Airflow初体验"
date:   2020-04-18 23:00:00 +0800
categories: Airflow
tags: Airflow-Start
excerpt: Airflow初体验
mathjax: true
typora-root-url: ../
---

# Airflow初体验

今天跟娃混了一天，都10点半了某娃还在磨蹭。。一整天都在叽里咕噜，叽里咕噜（包括写作业的时候），脑袋都发涨了。。看东西死活都看不进去

只好生搬硬凑一下作业 —— 看了半天没怎么看进去的airflow

Airflow is a platform to programmatically author, schedule and monitor workflows.

Airflow是一个调度工具，通过python来定义，可以用来定义工作流，据说是非常强大的。还有很赞的界面。

# DAG

这应该是属于airflow中最主要的概念了，Directed Acyclic Graphs（有向无循环图），用来定义一个完整的作业。airflow的scheduler会按照依赖关系来运行任务 —— 这点比cronjob啥的强大多了，DAG的重大作用是管理了作业的依赖关系，还有条件分支。

DAG中描述了需要运行的tasks，以及它们的关系

# DAG Run

当一个 DAG 满足它的调度时间，或者被外部触发时，就会产生一个 DAG Run。可以理解为由 DAG 实例化的实例

# Operators

Operator描述了DAG中一个具体的task具体要做的事，airflow有很多内置的operator：

- [`BashOperator`](https://airflow.apache.org/docs/stable/_api/airflow/operators/bash_operator/index.html#airflow.operators.bash_operator.BashOperator) - executes a bash command
- [`PythonOperator`](https://airflow.apache.org/docs/stable/_api/airflow/operators/python_operator/index.html#airflow.operators.python_operator.PythonOperator) - calls an arbitrary Python function
- [`EmailOperator`](https://airflow.apache.org/docs/stable/_api/airflow/operators/email_operator/index.html#airflow.operators.email_operator.EmailOperator) - sends an email
- [`SimpleHttpOperator`](https://airflow.apache.org/docs/stable/_api/airflow/operators/http_operator/index.html#airflow.operators.http_operator.SimpleHttpOperator) - sends an HTTP request
- [`MySqlOperator`](https://airflow.apache.org/docs/stable/_api/airflow/operators/mysql_operator/index.html#airflow.operators.mysql_operator.MySqlOperator), [`SqliteOperator`](https://airflow.apache.org/docs/stable/_api/airflow/operators/sqlite_operator/index.html#airflow.operators.sqlite_operator.SqliteOperator), [`PostgresOperator`](https://airflow.apache.org/docs/stable/_api/airflow/operators/postgres_operator/index.html#airflow.operators.postgres_operator.PostgresOperator), [`MsSqlOperator`](https://airflow.apache.org/docs/stable/_api/airflow/operators/mssql_operator/index.html#airflow.operators.mssql_operator.MsSqlOperator), [`OracleOperator`](https://airflow.apache.org/docs/stable/_api/airflow/operators/oracle_operator/index.html#airflow.operators.oracle_operator.OracleOperator), [`JdbcOperator`](https://airflow.apache.org/docs/stable/_api/airflow/operators/jdbc_operator/index.html#airflow.operators.jdbc_operator.JdbcOperator), etc. - executes a SQL command
- `Sensor` - an Operator that waits (polls) for a certain time, file, database row, S3 key, etc…

# Tasks

Task是operator的一个实例

# Task Instance

task的一次运行。task instance 有自己的状态，包括**"running", "success", "failed", "skipped", "up for retry"**等

# Task Relationships

DAGs中的不同Tasks之间可以有依赖关系，如 TaskA >> TaskB，表明TaskB依赖于TaskA

![image-20200418225615960](/../assets/images/image-20200418225615960.png)