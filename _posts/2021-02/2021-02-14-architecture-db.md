---
layout: post
title:  "高性能数据库集群"
date:   2021-02-14 23:01:00 +0800
categories: Architecture
tags: Architecture
excerpt: 高性能数据库集群
mathjax: true
typora-root-url: ../
---

# 高性能数据库集群

高性能数据库集群：

* 第一种方式是“读写分离”，其本质是将访问压力分散到集群中的多个节点，但是没有分散存储压力；
* 第二种方式是“分库分表”，既可以分散访问压力，又可以分散存储压力。

## 读写分离

读写分离的基本原理是将数据库读写操作分散到不同的节点：

* 数据库服务器搭建主从集群，一主一从、一主多从都可以。
* 数据库主机负责读写操作，从机只负责读操作。
* 数据库主机通过复制将数据同步到从机，每台数据库服务器都存储了所有的业务数据。
* 业务服务器将写操作发给数据库主机，将读操作发给数据库从机。

是“主从集群”，而不是“主备集群”。“从机”的“从”可以理解为“仆从”，仆从是要帮主人干活的，“从机”是需要提供读数据的功能的；而“备机”一般被认为仅仅提供备份功能，不提供访问功能。所以使用“主从”还是“主备”，是要看场景的，这两个词并不是完全等同的。

读写分离的实现逻辑并不复杂，但有两个细节点将引入设计复杂度：主从复制延迟和分配机制。

* 主从复制延迟解决办法

1. 写操作后的读操作指定发给数据库主服务器
2. 读从机失败后再读一次主机
3. 关键业务读写操作全部指向主机，非关键业务采用读写分离

* 分配机制

将读写操作区分开来，然后访问不同的数据库服务器，一般有两种方式：程序代码封装和中间件封装。

1. 程序代码封装：程序代码封装指在代码中抽象一个数据访问层（所以有的文章也称这种方式为“中间层封装”），实现读写操作分离和数据库服务器连接的管理
   * 实现简单，而且可以根据业务做较多定制化的功能。
   * 每个编程语言都需要自己实现一次，无法通用，如果一个业务包含多个编程语言写的多个子系统，则重复开发的工作量比较大
   * 故障情况下，如果主从发生切换，则可能需要所有系统都修改配置并重启。
2. 中间件封装：中间件封装指的是独立一套系统出来，实现读写操作分离和数据库服务器连接的管理。中间件对业务服务器提供SQL兼容的协议，业务服务器无须自己进行读写分离。对于业务服务器来说，访问中间件和访问数据库没有区别，事实上在业务服务器看来，中间件就是一个数据库服务器
   * 能够支持多种编程语言，因为数据库中间件对业务服务器提供的是标准SQL接口。
   * 数据库中间件要支持完整的SQL语法和数据库服务器的协议（例如，MySQL客户端和服务器的连接协议），实现比较复杂，细节特别多，很容易出现bug，需要较长的时间才能稳定。
   * 数据库中间件自己不执行真正的读写操作，但所有的数据库操作请求都要经过中间件，中间件的性能要求也很高。
   * 数据库主从切换对业务服务器无感知，数据库中间件可以探测数据库服务器的主从状态。例如，向某个测试表写入一条数据，成功的就是主机，失败的就是从机。

由于数据库中间件的复杂度要比程序代码封装高出一个数量级，一般情况下建议采用程序语言封装的方式，或者使用成熟的开源数据库中间件。如果是大公司，可以投入人力去实现

数据库中间件，因为这个系统一旦做好，接入的业务系统越多，节省的程序开发投入就越多，价值也越大。

## 分库分表

业务分库：业务分库指的是按照业务模块将数据分散到不同的数据库服务器

* join操作问题：业务分库后，原本在同一个数据库中的表分散到不同数据库中，导致无法使用SQL的join查询。
* 事务问题：原本在同一个数据库中不同的表可以在同一个事务中修改，业务分库后，表分散到不同的数据库中，无法通过事务统一修改。
* 成本问题：业务分库同时也带来了成本的代价，本来1台服务器搞定的事情，现在要3台，如果考虑备份，那就是2台变成了6台。

分表：将不同业务数据分散存储到不同的数据库服务器，能够支撑百万甚至千万用户规模的业务，但如果业务继续发展，同一业务的单表数据也会达到单台数据库服务器的处理瓶颈。

单表数据拆分有两种方式：垂直分表和水平分表。

单表进行切分后，是否要将切分后的多个表分散在不同的数据库服务器中，可以根据实际的切分效果来确定，并不强制要求单表切分为多表后一定要分散到不同数据库中。原因在于单表切分为多表后，新的表即使在同一个数据库服务器中，也可能带来可观的性能提升，如果性能能够满足业务要求，是可以不拆分到多台数据库服务器的，毕竟我们在上面业务分库的内容看到业务分库也会引入很多复杂性的问题；如果单表拆分为多表后，单台服务器依然无法满足性能要求，那就不得不再次进行业务分库的设计了。

* 垂直分表：垂直分表适合将表中某些不常用且占了大量空间的列拆分出去
* 水平分表：水平分表适合表行数特别大的表，有的公司要求单表行数超过5000万就必须进行分表，这个数字可以作为参考，但并不是绝对标准，关键还是要看表的访问性能。对于一些比较复杂的表，可能超过1000万就要分表了；而对于一些简单的表，即使存储数据超过1亿行，也可以不分表。但不管怎样，当看到表的数据量达到千万级别时，作为架构师就要警觉起来，因为这很可能是架构的性能瓶颈或者隐患。
* 路由：水平分表后，某条数据具体属于哪个切分后的子表，需要增加路由算法进行计算，这个算法会引入一定的复杂性。
* join操作：水平分表后，数据分散在多个表中，如果需要与其他表进行join查询，需要在业务代码或者数据库中间件中进行多次join查询，然后将结果合并。
* count操作：水平分表后，虽然物理上数据分散到多个表中，但某些业务逻辑上还是会将这些表当作一个表来处理。
* order by操作：水平分表后，数据分散到多个子表中，排序操作无法在数据库中完成，只能由业务代码或者数据库中间件分别查询每个子表中的数据，然后汇总进行排序。

在数据库遇到性能问题的时候，可以按如下顺序依次尝试：

1. 做硬件优化，例如从机械硬盘改成使用固态硬盘，当然固态硬盘不适合服务器使用，只是举个例子
2. 先做数据库服务器的调优操作，例如增加索引，oracle有很多的参数调整;
3. 引入缓存技术，例如Redis，减少数据库压力
4. 程序与数据库表优化，重构，例如根据业务逻辑对程序逻辑做优化，减少不必要的查询;
5. 在这些操作都不能大幅度优化性能的情况下，不能满足将来的发展，再考虑分库分表，也要有预估性