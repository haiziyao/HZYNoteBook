---
title: MySQL 相关命令
date: 2026-09-01
tags: [mysql]
description: 尚硅谷 MySQL 学习笔记
source: 尚硅谷 MySQL 课程笔记
---
## mysql

该mysql不是指mysql服务，而是指mysql的客户端工具。

语法：

```mysql
mysql [options] [database]
```

**连接选项**

```mysql
#参数：
    -u, --user=name 指定用户名
    -p, --password[=name] 指定密码
    -h, --host=name 指定服务器IP或域名
    -P, --port=# 指定连接端口
#示例 ：
    mysql -h 127.0.0.1 -P 3306 -u root -p
    mysql -h127.0.0.1 -P3306 -uroot -p密码
```

**执行选项**

```mysql
-e, --execute=name 执行SQL语句并退出
```

此选项可以在Mysql客户端执行SQL语句，而不用连接到MySQL数据库再执行，对于一些批处理脚本，这种方式尤其方便。

```mysql
#示例：
mysql -uroot -p db01 -e "select * from tb_book";
```

**自动补齐**

在`MySQL Command-Line Client`中启用自动补齐

```shell
mysql --auto-rehash
mysql -uroot -p<password> --database=dbName;
```

## mysqladmin

mysqladmin是一个执行管理操作的客户端程序。可以用它来检查服务器的配置和当前状态、创建并删除数据库等。

可以通过：mysqladmin --help指令查看帮助文档

```mysql
#示例 ：
    mysqladmin -uroot -p create 'test01';
    mysqladmin -uroot -p drop 'test01';
    mysqladmin -uroot -p version;
```

## mysqlshow

mysqlshow客户端对象查找工具，用来很快地查找存在哪些数据库、数据库中的表、表中的列或者索引。

语法：

```mysql
mysqlshow [options] [db_name [table_name [col_name]]]
```

参数：

```mysql
--count 显示数据库及表的统计信息（数据库，表均可以不指定）
-i 显示指定数据库或者指定表的状态信息
```

示例：

```mysql
# 查询每个数据库的表的数量及表中记录的数量
mysqlshow -uroot -p --count

# 查询dbname库中每个表中的字段数，及行数
mysqlshow -uroot -p dbname --count

# 查询dbname库中tablename表中的字段数，及行数
mysqlshow -uroot -p dbname tablename --count
```

## SHOW PROCESSLIST

mysql内部管理mysql进程

```mysql
SHOW PROCESSLIST;
```

