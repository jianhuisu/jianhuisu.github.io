---
title : 开发人员客串DBA必须掌握的mysql用户权限管理
categories : 
 - mysql 
tags :
	- MySQL
---
## 注意mysql的version

	mysql> select version();
	+-----------+
	| version() |
	+-----------+
	| 8.0.16    |
	+-----------+
	1 row in set (0.00 sec)

## 用户管理

注意在mysql的权限设计思想中,`host+user`被视为一个用户,而不能单单将一个user视为一个用户,
大白话就是上海的张三跟北京的张三可能不是一个人,所以要区别对待。

创建用户 
	
命令格式 `create user '用户名'@'主机IP' identified by '符合密码安全策略的密码'`
	
	mysql> create user 'select_user'@'%' identified by '4466xdebug_User';
	Query OK, 0 rows affected (0.00 sec)

删除用户

	mysql> drop user db_user_qq;
	Query OK, 0 rows affected (0.00 sec)


删除用户后,会将该用户关联的权限一并删除

	mysql> create user 'sujianhui'@'%' identified by 'yigeyouchangyoufuZAdemima_666';
	Query OK, 0 rows affected (0.00 sec)

	mysql> grant all privileges on study.* to sujianhui;
	Query OK, 0 rows affected (0.01 sec)

	mysql> show grants for sujianhui;
	+------------------------------------------------------+
	| Grants for sujianhui@%                               |
	+------------------------------------------------------+
	| GRANT USAGE ON *.* TO `sujianhui`@`%`                |
	| GRANT ALL PRIVILEGES ON `study`.* TO `sujianhui`@`%` |
	+------------------------------------------------------+
	2 rows in set (0.00 sec)

	mysql> drop user sujianhui;
	Query OK, 0 rows affected (0.01 sec)

	mysql> show grants for sujianhui;
	ERROR 1141 (42000): There is no such grant defined for user 'sujianhui' on host '%'

## 权限管理

新用户创建时默认不会分配任何权限，使用`select_user`用户查询`study`数据库

	mysql> use study
	ERROR 1044 (42000): Access denied for user 'select_user'@'%' to database 'study'	

赋予权限

命令格式 `grant 以逗号连接的权限字符串 on 数据库名.数据表名  to select_user`

	mysql> grant select on study.*  to select_user;
	Query OK, 0 rows affected (0.00 sec)

回收权限 权限不存在则报错

	mysql> revoke  select on study.* from select_user;

	
查看一个用户具有哪些权限
	
	mysql> show grants for select_user;
	+-----------------------------------------+
	| Grants for select_user@%                |
	+-----------------------------------------+
	| GRANT USAGE ON *.* TO `select_user`@`%` |
	+-----------------------------------------+
	1 row in set (0.00 sec)

`USAGE`表示没有任何权限
	
Tips:	mysql8.x修改完权限以后,立即生效,但是为了保险起见,还是手动刷新一下`flush privileges`
(mysql8.0 之前的版本貌似不手动刷新`flush privileges`权限不会生效,具体是哪个版本修改的因为时间原因,我就没有去测试)


## 如何查看`all privileges`都包含哪些权限

	mysql> grant all privileges on study.* to select_user;
	Query OK, 0 rows affected (0.01 sec)

	mysql> show grants for select_user;
	+--------------------------------------------------------+
	| Grants for select_user@%                               |
	+--------------------------------------------------------+
	| GRANT USAGE ON *.* TO `select_user`@`%`                |
	| GRANT ALL PRIVILEGES ON `study`.* TO `select_user`@`%` |
	| GRANT SELECT ON `zeus`.* TO `select_user`@`%`          |
	+--------------------------------------------------------+
	3 rows in set (0.00 sec)

	mysql> revoke select on study.* from select_user;
	Query OK, 0 rows affected (0.01 sec)

	
	mysql> mysql> show grants for select_user\G
	*************************** 1. row ***************************
	Grants for select_user@%: GRANT USAGE ON *.* TO `select_user`@`%`
	*************************** 2. row ***************************
	Grants for select_user@%: GRANT INSERT, UPDATE, DELETE, CREATE, DROP, REFERENCES, INDEX, ALTER, CREATE TEMPORARY TABLES,
	LOCK TABLES, EXECUTE, CREATE VIEW, SHOW VIEW, CREATE ROUTINE, ALTER ROUTINE, EVENT, TRIGGER ON `study`.* TO `select_user`@`%`
	*************************** 3. row ***************************
	Grants for select_user@%: GRANT SELECT ON `zeus`.* TO `select_user`@`%`
	3 rows in set (0.00 sec)

将第二行的权限与`SELECT`权限合并,则为`all privileges`所包含的所有权限
	
	INSERT
	UPDATE
	DELETE
	CREATE
	DROP
	REFERENCES
	INDEX
	ALTER
	CREATE TEMPORARY TABLES
	LOCK TABLES
	EXECUTE
	CREATE VIEW
	SHOW VIEW
	CREATE ROUTINE // 创建存储过程
	ALTER ROUTINE  // 修改存储过程
	EVENT
    TRIGGER
	
如果没有被赋予这些权限,意味着不能在SQL中使用这些关键字,一些函数或者其它关键字的使用则没有限制

Tips:这个`all privileges`并不是最全的,创建过一个拥有主从复制权限的用户
	
	mysql> show grants for slave_account;
	+-------------------------------------------------------+
	| Grants for slave_account@%                            |
	+-------------------------------------------------------+
	| GRANT REPLICATION SLAVE ON *.* TO `slave_account`@`%` |
	+-------------------------------------------------------+
	1 row in set (0.00 sec)

`REPLICATION`并不在上述列表,所以这个`all privileges`应该针对的是数据库使用群体,而不是数据库的管理群体

## 零散知识点

user表中host列的值的意义

- %              匹配所有主机,也可以这样使用`192.168.126.%`
- localhost      localhost不会被解析成IP地址，直接通过UNIXsocket连接
- 127.0.0.1      会通过TCP/IP协议连接，并且只能在本机访问；
- ::1            ::1就是兼容支持ipv6的，表示同ipv4的127.0.0.1

## 参考资料

感谢作者 https://www.cnblogs.com/fslnet/p/3143344.html

