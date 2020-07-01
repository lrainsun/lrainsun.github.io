---
layout: post
title:  "mysql monitoring"
date:   2020-07-02 23:00:00 +0800
categories: Linux
tags: Linux-Service
excerpt: mysql monitoring
mathjax: true
typora-root-url: ../
---

# mysql monitoring

## 吞吐量

* QPS（数据库每秒处理的请求数量） 相关的包括了三个监控项：Queries、Questions、Com_select，通常用Com_select表示

  * Queries：mysql系统接收的查询的次数，包括存储过程内部的查询
  * Questions：mysql系统接收查询的次数，但是不包括存储过程内部的查询（如果不使用 prepared statements ，那么两者的区别是 Questions 会将存储过程作为一个语句；而 Queries 会统计存储过程中的各个执行的语句）
  * Com_select：查询操作的次数

  QPS=QUESTIONS/UPTIME

* TPS（数据库每秒处理的事务数量）

  * Com_insert：插入操作的次数
  * Com_update：更新操作的次数
  * Com_delete：删除操作的次数

  TPS有两种计算方式：

  TPS = (COM_COMMIT + COM_ROLLBACK)/UPTIME

  TPS=Com_insert,Com_update,Com_delete之和

![image-20200701102534894](/../assets/images/image-20200701102534894.png)

```text
Questions        已执行的由客户端发出的语句
QPS              一台服务器每秒能够相应的查询次数，是对一个特定的查询服务器在规定时间内所处理流量多少的衡量标准
TPS              一个事务是指一个客户机向服务器发送请求然后服务器做出反应的过程。客户机在发送请求时开始计时，收到服务器响应后结束计时，以此来计算使用的时间和完成的事务个数
```

## 性能指标

* 连接数

客户端连接情况相当重要，因为一旦可用连接耗尽，新的客户端连接就会遭到拒绝

```mysql
[root@ci81hf1cmp001 ocpadmin]# mysql -e "SHOW VARIABLES LIKE 'max_connections';"
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections | 16384 |
+-----------------+-------+
```

可以通过 Threads_connected 查看，监控该指标与先前设置的连接限制，可以确保服务器拥有足够的容量处理新的连接。

通过 Threads_running 指标，可以查看正在处理请求的线程，可以用来判断那些连接被占用但是却没有处理任何请求。

如果达到 max_connections 就会拒绝新的连接请求，Connection_errors_max_connections 指标就会开始增加，同时，追踪所有失败连接尝试的 Aborted_connects 指标也会开始增加。

```text
Threads_connected                   已经建立的连接
Threads_running                     正在运行的连接
Connection_errors_internal          由于服务器内部本身导致的错误
Aborted_connects                    尝试与服务器建立连接但是失败的次数
Connection_errors_max_connections   由于到达最大连接数导致的错误
```

* 缓存使用情况

InnoDB 使用一片内存区域作为缓冲区，用来缓存数据表与索引数据，缓冲区太小可能会导致数据库性能下滑，磁盘 I/O 攀升。

```mysql
[root@ci81hf1cmp001 ocpadmin]# mysql -e "SHOW GLOBAL VARIABLES LIKE 'innodb_buffer_pool_chunk_size';"

[root@ci81hf1cmp001 ocpadmin]# mysql -e "SHOW GLOBAL VARIABLES LIKE 'innodb_buffer_pool_instances';"
+------------------------------+-------+
| Variable_name                | Value |
+------------------------------+-------+
| innodb_buffer_pool_instances | 1     |
+------------------------------+-------+

[root@ci81hf1cmp001 ocpadmin]# mysql -e "SHOW GLOBAL VARIABLES LIKE 'innodb_buffer_pool_size';"
+-------------------------+-----------+
| Variable_name           | Value     |
+-------------------------+-----------+
| innodb_buffer_pool_size | 134217728 |
+-------------------------+-----------+
```

如果 innodb_buffer_pool_chunk_size 查询没有返回结果，则表示在你使用的 MySQL 版本中此参数无法更改，其值为 128 MiB，实际参数为 innodb_buffer_pool_size 。

Innodb_buffer_pool_read_requests 记录了读取请求的数量，而 Innodb_buffer_pool_reads 记录了缓冲池无法满足，因而只能从磁盘读取的请求数量，也就是说，如果 Innodb_buffer_pool_reads 的值开始增加，意味着数据库性能大有问题。

缓存的使用率和命中率可以通过如下方法计算：

```
(Innodb_buffer_pool_pages_total - Innodb_buffer_pool_pages_free) /
    Innodb_buffer_pool_pages_total * 100%

(Innodb_buffer_pool_read_requests - Innodb_buffer_pool_reads) /
    Innodb_buffer_pool_read_requests * 100%
```

如果数据库从磁盘进行大量读取，而缓冲池还有许多闲置空间，这可能是因为缓存最近才清理过，还处于预热阶段。

```
# HELP mysql_global_status_buffer_pool_pages Innodb buffer pool pages by state.
# TYPE mysql_global_status_buffer_pool_pages gauge
mysql_global_status_buffer_pool_pages{state="data"} 8058
mysql_global_status_buffer_pool_pages{state="free"} 0
mysql_global_status_buffer_pool_pages{state="misc"} 133
mysql_global_status_buffer_pool_pages{state="old"} 2955
```

pages total是这些加起来

```text
Innodb_buffer_pool_pages_total          BP中总页面数
Buffer pool utilization                 BP中页面的使用率
Innodb_buffer_pool_read_requests        BP的读请求
Innodb_buffer_pool_reads                需要读取磁盘的请求数
```

![image-20200701143853372](/../assets/images/image-20200701143853372.png)

# 高可用性指标

* 慢查询
* 死锁

![image-20200701150658762](/../assets/images/image-20200701150658762.png)

* 打开文件数

Opened_files：从上一次冲洗mysqld到现在，打开过的文件数

Open_files：当前打开的文件数

![image-20200701154630400](/../assets/images/image-20200701154630400.png)

