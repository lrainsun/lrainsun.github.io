---
layout: post
title:  "mysql修改密码"
date:   2020-03-18 23:00:00 +0800
categories: Linux
tags: Linux-Service
excerpt: mysql修改密码
mathjax: true
typora-root-url: ../
---

# mysql修改密码

今天我们碰到个问题，mysql里user的密码不对，要修改密码

用这样一条命令可以修改密码

```mysql
UPDATE mysql.user SET Password=PASSWORD('password') WHERE User='prometheus' AND Host='%';
```

可是试来试去死活还是不行，最后发现，修改完之后，还要flush一下

```mysql
FLUSH PRIVILEGES;
```

# privileges生效

If you modify the grant tables directly using statements such as [`INSERT`](https://dev.mysql.com/doc/refman/8.0/en/insert.html), [`UPDATE`](https://dev.mysql.com/doc/refman/8.0/en/update.html), or [`DELETE`](https://dev.mysql.com/doc/refman/8.0/en/delete.html) (which is not recommended), the changes have no effect on privilege checking until you either tell the server to reload the tables or restart it. Thus, if you change the grant tables directly but forget to reload them, the changes have *no effect* until you restart the server. This may leave you wondering why your changes seem to make no difference!

If the [**mysqld**](https://dev.mysql.com/doc/refman/8.0/en/mysqld.html) server is started without the [`--skip-grant-tables`](https://dev.mysql.com/doc/refman/8.0/en/server-options.html#option_mysqld_skip-grant-tables) option, it reads all grant table contents into memory during its startup sequence. The in-memory tables become effective for access control at that point.

If you modify the grant tables indirectly using an account-management statement, the server notices these changes and loads the grant tables into memory again immediately.

A grant table reload affects privileges for each existing client session as follows:

- Table and column privilege changes take effect with the client's next request.

- Database privilege changes take effect the next time the client executes a `USE *`db_name`*` statement.

  Note

  Client applications may cache the database name; thus, this effect may not be visible to them without actually changing to a different database.

- Global privileges and passwords are unaffected for a connected client. These changes take effect only in sessions for subsequent connections.

# grant tables

The `mysql` system database includes several grant tables that contain information about user accounts and the privileges held by them. This section describes those tables. 

```
These mysql database tables contain grant information:

user: User accounts, global privileges, and other nonprivilege columns.

db: Database-level privileges.

tables_priv: Table-level privileges.

columns_priv: Column-level privileges.

procs_priv: Stored procedure and function privileges.

proxies_priv: Proxy-user privileges.
```

# cluster

Mysql是三节点cluster，我们发现一台上面改了之后，只有一台生效了，其他还要重新改

```
Currently replication works only with the InnoDB storage engine. Any writes to tables of other types, including system (mysql.*) tables are not replicated (this limitation excludes DDL statements such as CREATE USER, which implicitly modify the mysql.* tables — those are replicated). There is however experimental support for MyISAM - see the wsrep_replicate_myisam system variable)
```

所以貌似除了create user操作之外，其他都不会cluster间同步