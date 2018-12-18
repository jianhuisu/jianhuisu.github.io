---
title ： SQL基础
---


## SQL简介

**SQL（structure query language）** 结构化查询语言，是一种对 基于关系模型构建的数据库 进行数据检索的语言工具，MySQL 是标准SQL的扩展。

## SQL分类

- DDL语句 data defintion language 数据定义语句 create、drop alter
- DML语句 data manipolution language 数据操纵语句 insert delete update select（增删改查）
- DCL语句 data control language  数据控制语句 控制访问级别等 grant revoke

### DDL语句

创建数据库

	create database db_name

删除数据库

	drop database db_name

创建数据表

	create table table_name

删除数据表

	drop table table_name

创建索引

	create [index,upique index] 索引名 on 表名（列名）
	mysql> create index `index` on ttttt(name);

修改数据表中列名

	alter table `table_name` change column `列2` `column_2` INT(11) NOT NULL DEFAULT '0' after `id`;

数据表中新加列

	alter table `table_name` add 列名 列类型 [first,after 列名]

数据表中删除列

	alter table `table_name` drop 列名；

### DML语句

插入单条数据

	insert into xxoo1(`name`,`sex`) values('machine',1)

批量插入

	insert into xxoo1(`name`,`sex`) values('machine1',1),('machine2',2),(),()...

使用结果集插入

	insert into xxoo1(`name`,`sex`) select `name`,`sex` from xxoo1

REPLACE INTO

	replace into xxoo1(`name`,`sex`) values('machine',2)
 
replace into 语句实质为 删除原数据，插入新数据 两个操作结合,通过对比下面`column_2`值为0的数据项可验证

	mysql> select * from replace_table;
	+----+----------+----------+
	| id | column_2 | name     |
	+----+----------+----------+
	|  1 |        0 | NULL     |
	|  3 |     3333 | machine3 |
	|  4 |      222 | machine4 |
	+----+----------+----------+
	3 rows in set (0.00 sec)
	
	mysql> replace into replace_table(name) values('machine4');
	Query OK, 2 rows affected (0.03 sec)
	
	mysql> select * from replace_table;
	+----+----------+----------+
	| id | column_2 | name     |
	+----+----------+----------+
	|  3 |     3333 | machine3 |
	|  4 |      222 | machine4 |
	|  5 |        0 | machine4 |
	+----+----------+----------+
	3 rows in set (0.00 sec)

删除表中数据，不归零主键

	delete from `table_name` 条件

清空表中数据 ，重新归零主键

	truncate table 表名  

例如

	mysql> show create table ttttt\G
	*************************** 1. row ***************************
	       Table: ttttt
	Create Table: CREATE TABLE `ttttt` (
	  `id` int(11) NOT NULL AUTO_INCREMENT,
	  `column_2` int(11) NOT NULL DEFAULT '0',
	  `name` char(50) DEFAULT NULL,
	  PRIMARY KEY (`id`),
	  UNIQUE KEY `column_2` (`column_2`),
	  KEY `index` (`name`)
	) ENGINE=InnoDB AUTO_INCREMENT=7 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
	1 row in set (0.00 sec)


	mysql> select * from ttttt;
	+----+----------+----------+
	| id | column_2 | name     |
	+----+----------+----------+
	|  6 |      222 | machine4 |
	|  7 |     3333 | machine3 |
	+----+----------+----------+
	2 rows in set (0.00 sec)
	
	mysql> truncate table ttttt;
	Query OK, 0 rows affected (0.14 sec)
	
	mysql> select * from ttttt;
	Empty set (0.00 sec)

更新字段值

	update `table_name` set 列名=列值  条件

**查询**
 
	select `id`,`colunm_2` from `replace_table` 条件

聚合函数

	select * from compile_foo where date=201811 group by businessUnit
	

> 非严格模式下:使用聚合函数对结果集进行分类时，后面的子字段保留第一次匹配的结果.严格模式下sql_mode=only_full_group_by。对于GROUP BY聚合操作，如果在SELECT中的列，没有在GROUP BY中出现，那么这个SQL是不合法的，因为列不在GROUP BY从句中，所以对于设置了这个mode的数据库，在使用group by 的时候，就要用MAX(),SUM(),ANT_VALUE()这种聚合函数，才能完成GROUP BY 的聚合操作。
> 

连接查询分为内连接与外链接

内连接 只显示相互匹配的数据

	select qa_type.name,qa_content_new.title from qa_type,qa_content_new where qa_content_new.type_id=qa_type.id

外链接 会显示未匹配的部分

	左查询  left join on
	右查询  rignt join on
	       inner join

结果集联合  union （去重 distinct）| union all  

	select type_id from qa_content_new union all select id from qa_type
	select type_id from qa_content_new union select id from qa_type


### DCL语句

	略

## 查看帮助

在命令行中使用 ？ + 关键字可快速检索mysql自带帮助文档

eg.1

	mysql> ? contents
	You asked for help about help category: "Contents"
	For more information, type 'help <item>', where <item> is one of the following
	categories:
	   Account Management
	   ...

eg.2 
	
	mysql> ? insert
	Name: 'INSERT'
	Description:
	Syntax:
		...
	
eg.3

	mysql> ? int
	Name: 'INT'
	Description:
	INT[(M)] [UNSIGNED] [ZEROFILL]
	
	A normal-size integer. The signed range is -2147483648 to 2147483647.
	The unsigned range is 0 to 4294967295.
	
	URL: http://dev.mysql.com/doc/refman/8.0/en/numeric-type-overview.html

## 其它常用语句

查看当前所在数据库

	mysql> select database();
	+------------+
	| database() |
	+------------+
	| qq         |
	+------------+

切换数据库
	
	use db_name;

查看建表语句
	
	create table table_name 或 desc table_name
	

