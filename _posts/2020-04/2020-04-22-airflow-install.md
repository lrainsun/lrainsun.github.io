---
layout: post
title:  "Airflow安装"
date:   2020-04-22 23:00:00 +0800
categories: Airflow
tags: Airflow-Start
excerpt: Airflow安装
mathjax: true
typora-root-url: ../

---

# Airflow开发环境安装

安装pyenv和pyenv-virtualenv

```shell
git clone https://github.com/pyenv/pyenv.git ~/.pyenv
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bash_profile
echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bash_profile
echo -e 'if command -v pyenv 1>/dev/null 2>&1; then\n  eval "$(pyenv init -)"\nfi' >> ~/.bash_profile
exec "$SHELL"
```

restart shell

遇到一个问题

```shell
[root@rain-working-server ~]# pyenv install 3.6.0
python-build: TMPDIR=/tmp cannot hold executables (partition possibly mounted with `noexec`)
[root@rain-working-server ~]# sudo mount -o remount,exec /tmp
```

```shell
pyenv install 3.6.0
```

```shell
git clone https://github.com/pyenv/pyenv-virtualenv.git $(pyenv root)/plugins/pyenv-virtualenv
echo 'eval "$(pyenv virtualenv-init -)"' >> ~/.bash_profile
exec "$SHELL"
```

vim ~/.bashrc

```shell
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"
```

```shell
pyenv virtualenv 3.6.0 airflow
```

# 安装airflow

```shell
pyenv activate airflow
pip install apache-airflow
```

我们要并行处理，所以需要安装其他database，我们装mysql

```shell
yum install mysql mysql-server
service mariadb start

pip install 'apache-airflow[mysql]'
```

# 配置mysql

设置root密码，并创建airflow user

```mariadb
SET PASSWORD=PASSWORD("356P@ssword_356");
FLUSH PRIVILEGES; 

CREATE DATABASE airflow; 
GRANT all privileges on airflow.* TO 'airflow_user'@'localhost'  IDENTIFIED BY '123airflow_Password'; 
FLUSH PRIVILEGES; 
```

修改airflow配置，vim ~/airflow/airflow.cfg

```shell
sql_alchemy_conn = mysql://airflow_user:123airflow_Password@localhost/airflow
//dialect+driver://username:password@host:port/database
```

```mariadb
MariaDB [(none)]> use airflow
Database changed
MariaDB [airflow]> show tables;
Empty set (0.000 sec)
```

初始化数据库

```shell
airflow initdb
```

遇到一个问题

> Exception: Global variable explicit_defaults_for_timestamp needs to be on (1) for mysql

在 `/etc/my.cnf`增加配置

```
[mysqld]
explicit_defaults_for_timestamp = 1
```

```shell
service mariadb restart
```

db init之后，会生成这么多表

```mariadb
MariaDB [(none)]> use airflow
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [airflow]> show tables;
+-------------------------------+
| Tables_in_airflow             |
+-------------------------------+
| alembic_version               |
| chart                         |
| connection                    |
| dag                           |
| dag_code                      |
| dag_pickle                    |
| dag_run                       |
| dag_tag                       |
| import_error                  |
| job                           |
| known_event                   |
| known_event_type              |
| kube_resource_version         |
| kube_worker_uuid              |
| log                           |
| rendered_task_instance_fields |
| serialized_dag                |
| sla_miss                      |
| slot_pool                     |
| task_fail                     |
| task_instance                 |
| task_reschedule               |
| users                         |
| variable                      |
| xcom                          |
+-------------------------------+
25 rows in set (0.001 sec)
```

# 启动airflow

修改配置文件`~/airflow/airflow.cfg`

```shell
executor = LocalExecutor
```

```shell
(airflow) [root@rain-working-server ~]# airflow webserver --debug
 ____    |__( )_________  __/__  /________      __
____  /| |_  /__  ___/_  /_ __  /_  __ \_ | /| / /
___  ___ |  / _  /   _  __/ _  / / /_/ /_ |/ |/ /
 _/_/  |_/_/  /_/    /_/    /_/  \____/____/|__/
Starting the web server on port 8080 and host 0.0.0.0.
[2020-04-22 05:31:06,420] {__init__.py:51} INFO - Using executor LocalExecutor
[2020-04-22 05:31:06,421] {dagbag.py:396} INFO - Filling up the DagBag from /root/airflow/dags
 * Serving Flask app "airflow.www.app" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
[2020-04-22 05:31:06,673] {_internal.py:122} INFO -  * Running on http://0.0.0.0:8080/ (Press CTRL+C to quit)
[2020-04-22 05:31:06,673] {_internal.py:122} INFO -  * Restarting with stat
  ____________       _____________
 ____    |__( )_________  __/__  /________      __
____  /| |_  /__  ___/_  /_ __  /_  __ \_ | /| / /
___  ___ |  / _  /   _  __/ _  / / /_/ /_ |/ |/ /
 _/_/  |_/_/  /_/    /_/    /_/  \____/____/|__/
Starting the web server on port 8080 and host 0.0.0.0.
[2020-04-22 05:31:08,436] {__init__.py:51} INFO - Using executor LocalExecutor
[2020-04-22 05:31:08,437] {dagbag.py:396} INFO - Filling up the DagBag from /root/airflow/dags
[2020-04-22 05:31:08,691] {_internal.py:122} WARNING -  * Debugger is active!
[2020-04-22 05:31:08,692] {_internal.py:122} INFO -  * Debugger PIN: 212-311-139
```

启动scheduler

```shell
(airflow) [root@rain-working-server ~]# airflow scheduler
  ____________       _____________
 ____    |__( )_________  __/__  /________      __
____  /| |_  /__  ___/_  /_ __  /_  __ \_ | /| / /
___  ___ |  / _  /   _  __/ _  / / /_/ /_ |/ |/ /
 _/_/  |_/_/  /_/    /_/    /_/  \____/____/|__/
[2020-04-22 05:33:25,944] {__init__.py:51} INFO - Using executor LocalExecutor
[2020-04-22 05:33:25,983] {scheduler_job.py:1346} INFO - Starting the scheduler
[2020-04-22 05:33:25,983] {scheduler_job.py:1354} INFO - Running execute loop for -1 seconds
[2020-04-22 05:33:25,985] {scheduler_job.py:1355} INFO - Processing each file at most -1 times
[2020-04-22 05:33:25,985] {scheduler_job.py:1358} INFO - Searching for files in /root/airflow/dags
[2020-04-22 05:33:25,998] {scheduler_job.py:1360} INFO - There are 24 files in /root/airflow/dags
[2020-04-22 05:33:26,105] {scheduler_job.py:1411} INFO - Resetting orphaned tasks for active dag runs
[2020-04-22 05:33:26,122] {dag_processing.py:556} INFO - Launched DagFileProcessorManager with pid: 9339
[2020-04-22 05:33:26,130] {settings.py:54} INFO - Configured default timezone <Timezone [UTC]>
```

# 安装加密库

> **Warning:** Connection passwords are stored in plaintext until you  install the Python "cryptography" library. You can find installation  instructions here: https://cryptography.io/en/latest/installation/. Once installed, instructions for creating an encryption key will be displayed the next time you import Airflow. 

```shell
pip install cryptography
```

