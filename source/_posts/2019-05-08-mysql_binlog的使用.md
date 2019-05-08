---
title : binlog的使用
categories : 
 - mysql 
tags :
	- MySQL
---

## binlog的用途
	
binlog是记录所有数据库表结构变更（例如CREATE、ALTER TABLE…）以及表数据修改（INSERT、UPDATE、DELETE…）的二进制日志。
binlog不会记录SELECT和SHOW这类操作，因为这类操作对数据本身并没有修改，但你可以通过查询通用日志来查看MySQL执行过的所有语句。

二进制日志包括两类文件：

- 索引文件（文件名后缀为.index）用于记录哪些日志文件正在被使用
- 日志文件（文件名后缀为.00000*）记录数据库所有的DDL和DML(除了数据查询语句)语句事件。

binlog的主要应用场景

1. 审计 审计数据服务是否正常
1. 复制 数据迁移
1. 恢复 宕机、误删后数据恢复
	
## 如何启用binlog

vim /etc/my.cnf
	
	[mysqld]
	...
	# 设置数据库的 server-id 设置集群时使用
	server-id=1
	
	# 开启binlog 并设置binlog的前缀
	log-bin=mysql-bin
	
	#对哪些库的操作进行记录
	binlog-do-db=study
	
	#忽略对指定库的操作记录
	binlog-ignore-db=mysql


## 如何查看binlog的相关设置

	mysql> show global variables like "%binlog%";
	+------------------------------------------------+----------------------+
	| Variable_name                                  | Value                |
	+------------------------------------------------+----------------------+
	| binlog_cache_size                              | 32768                |
	 ...
	| binlog_format                                  | ROW                  |
	| binlog_expire_logs_seconds                     | 2592000              |
	 ...
	| max_binlog_cache_size                          | 18446744073709547520 |
	| max_binlog_size                                | 1073741824           |
	| max_binlog_stmt_cache_size                     | 18446744073709547520 |
	| sync_binlog                                    | 1                    |
	+------------------------------------------------+----------------------+
	27 rows in set (0.00 sec)

重要配置项
	
	binlog_format(binlog常见格式):
	
			statement   : sql语句(当sql中包含now()、uuid()等函数,会产生复制不准确问题)
			row(推荐)   : 数据变更
			mixed       : statement + row 模式的混合
			
	sync_binlog: sync_binlog=N，代表每N次事务提交时就会将缓存中的日志持久化到文件,一般设置为 1
	binlog_expire_logs_seconds   binlog的保存时间 默认为一个月
	
## 如何查看binlog内容

binlog文件本身是二进制文件,打开后无法直接阅读,需要借助mysql提供的阅读工具`mysqlbinlog`

查看statement格式(显示的基本都是普通sql语句)

	mysqlbinlog mysql-bin.00001
		
查看rows格式(格式化以后估计也看不太懂)
	
	mysqlbinlog -vv mysql-bin.00001

## binlog文件越来越多,如何清理

有时候binlog还没等到过期,就已经很多,此时需要我们手动删除,下面是三种安全的删除方式

1. `reset master` 删除所有binlog日志文件（除mysql-bin.index文件）
1. `purge master logs to mysql-bin.******` 将******编号之前的binlog日志文件删除
1. `purge master logs before 'yyyy-mm-dd hh24:mi:ss'` 删除 yyyy-mm-dd hh24:mi:ss日期之前产生的所有日志

如果想使用`rm`直接删除,注意不要将最新binlog文件删除

Tips:
	如果mysql服务器是单机,直接删除日志没有太大问题,但是如果是集群,一定要确定主从数据的一致性以后再操作

## 参考资料

强烈推荐(本笔记的知识点绝大部分都是对该文的整理) https://www.cnblogs.com/rjzheng/p/9721765.html	

	