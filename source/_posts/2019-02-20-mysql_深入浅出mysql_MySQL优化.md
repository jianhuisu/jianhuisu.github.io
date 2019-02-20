---
title : MySQL优化
tags : 
	- MySQL
	- 优化
---

MySQL的优化主要分为三个方面

1. SQL语句
1. 应用方面
1. MySQL服务架构

本篇笔记主要从宏观上对SQL调优的步骤进行梳理，便于转化为自己的知识体系，因为篇幅限制，文中涉及到的重要的知识点会后续更新中补充

## Part.1 SQL语句优化

### step.1 对数据服务有基本的了解

show global status like "Com_select%"

	+-------------------------------------+-------+
	| Variable_name                       | Value |
	+-------------------------------------+-------+
	| Com_commit                          | 0     |	
	| Com_delete                          | 0     |
	| Com_delete_multi                    | 0     |
	| Com_insert                          | 0     |
	| Com_insert_select                   | 0     |
	| Com_rollback                        | 0     |
	| Com_select                          | 2     |
	| Com_update                          | 0     |
	...
	+-------------------------------------+-------+

Com_xxx代表xxx语句执行频率，show global 代表统计从mysql进程启动开始，show session代表统计本次会话，通过观察DML语句的执行频率，可以了解数据库服务的查询、插入、事务等使用占比。

查看innodb存储引擎相关操作统计

	mysql> show global status like "innodb_rows%";
	+----------------------+-------+
	| Variable_name        | Value |
	+----------------------+-------+
	| Innodb_rows_deleted  | 0     |
	| Innodb_rows_inserted | 0     |
	| Innodb_rows_read     | 4886  |
	| Innodb_rows_updated  | 313   |
	+----------------------+-------+
	4 rows in set (0.00 sec)

慢查询统计次数

	mysql> show global status like "slow_queries%" 

系统连接次数

	mysql> show global status like "Connections%"  
	+---------------+-------+
	| Variable_name | Value |
	+---------------+-------+
	| Connections   | 8     |
	+---------------+-------+
	1 row in set (0.00 sec)

	使用脚本循环连接

	<?php

		for($i = 0; $i < 10 ;$i++)
		{
		    $con = mysqli_connect('172.16.125.253','root','password','qq');
		}
	?>
	
	再次查询连接次数
	mysql> show global status like "Connections%";
	+---------------+-------+
	| Variable_name | Value |
	+---------------+-------+
	| Connections   | 18    |
	+---------------+-------+
	1 row in set (0.00 sec)

 系统运行时间

	mysql> show global status like "Uptime%"      

备注：

- show [global|session] status    查询mysql运行时可变变量
- show [global|session] variables 查询mysql服务预定义变量

### step.2 定位执行缓慢的SQL 

#### 慢查询日志

开启慢查询日志

	[mysqld]
	...

	slow_query_log = ON
    slow_query_log_file = /var/lib/mysql/vagrant-centos6.5-slow.log
    long_query_time = 2

查看慢查询日志配置

	mysql> show variables like "%slow%";
	+---------------------------+-------------------------------------------+
	| Variable_name             | Value                                     |
	+---------------------------+-------------------------------------------+
	| log_slow_admin_statements | OFF                                       |
	| log_slow_slave_statements | OFF                                       |
	| slow_launch_time          | 2                                         |
	| slow_query_log            | ON                                        |
	| slow_query_log_file       | /var/lib/mysql/vagrant-centos6.5-slow.log |
	+---------------------------+-------------------------------------------+
	5 rows in set (0.01 sec)
	
	mysql> show variables like 'long_query_time';
	+-----------------+----------+
	| Variable_name   | Value    |
	+-----------------+----------+
	| long_query_time | 2.000000 |
	+-----------------+----------+
	1 row in set (0.01 sec)


构建一个查询超过2s的SQL

	mysql> select sleep(5);
	+----------+
	| sleep(5) |
	+----------+
	|        0 |
	+----------+
	1 row in set (5.00 sec)

查看日志

	# Time: 2019-02-20T03:37:01.385800Z
	# User@Host: root[root] @ localhost []  Id:     8
	# Query_time: 5.000303  Lock_time: 0.000000 Rows_sent: 1  Rows_examined: 0
	SET timestamp=1550633821;
	select sleep(5);  // 这是实际执行的SQL


#### show [full] processlist 实时查看

慢查询日志可以记录已经执行完成执行效率较低的SQL，如果现在MySQL很卡，如果查看MySQL的负载呢

Connection 19线程有SQL正在执行时，通过`show full processlist`可以查看线程执行状态 是否锁表 等

	mysql> show full processlist;
	+----+-----------------+----------------------+------+---------+------+------------------------+------------------+
	| Id | User            | Host                 | db   | Command | Time | State                  | Info             |
	+----+-----------------+----------------------+------+---------+------+------------------------+------------------+
	|  4 | event_scheduler | localhost            | NULL | Daemon  | 5529 | Waiting on empty queue | NULL             |
	|  8 | root            | localhost            | qq   | Query   |    0 | starting               | show processlist |
	| 19 | root            | 172.16.125.191:51591 | qq   | Query   |    1 | User sleep             | select sleep(5)  |
	+----+-----------------+----------------------+------+---------+------+------------------------+------------------+
	3 rows in set (0.00 sec)
	
Connection 19 线程执行结束后，释放状态描述

	mysql> show full processlist;
	+----+-----------------+----------------------+------+---------+------+------------------------+------------------+
	| Id | User            | Host                 | db   | Command | Time | State                  | Info             |
	+----+-----------------+----------------------+------+---------+------+------------------------+------------------+
	|  4 | event_scheduler | localhost            | NULL | Daemon  | 5539 | Waiting on empty queue | NULL             |
	|  8 | root            | localhost            | qq   | Query   |    0 | starting               | show processlist |
	| 19 | root            | 172.16.125.191:51591 | qq   | Sleep   |   11 |                        | NULL             |
	+----+-----------------+----------------------+------+---------+------+------------------------+------------------+
	3 rows in set (0.00 sec)

当然，如果条件允许，可以直接杀掉这个线程

	mysql> show full processlist;
	+----+-----------------+----------------------+------+---------+------+------------------------+-----------------------+
	| Id | User            | Host                 | db   | Command | Time | State                  | Info                  |
	+----+-----------------+----------------------+------+---------+------+------------------------+-----------------------+
	|  4 | event_scheduler | localhost            | NULL | Daemon  | 6092 | Waiting on empty queue | NULL                  |
	|  8 | root            | localhost            | qq   | Query   |    0 | starting               | show full processlist |
	| 19 | root            | 172.16.125.191:51591 | qq   | Query   |    4 | User sleep             | select sleep(100)     |
	+----+-----------------+----------------------+------+---------+------+------------------------+-----------------------+
	3 rows in set (0.00 sec)
	
	mysql> kill query 19;
	Query OK, 0 rows affected (0.00 sec)

备注：

- MySQL是单进程，多线程 进程挂了，整个服务也就挂了
- Oracle在linux上默认是多进程，多进程的好处是一个进程崩溃不会影响另外的进程
	
### step.3 分析SQL
	
使用工具分析低效SQL的执行计划

#### explain/desc

	mysql> explain select * from region left join business on region.business_id=business.business_id;
	+----+-------------+----------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
	| id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                              |
	+----+-------------+----------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
	|  1 | SIMPLE      | region   | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    3 |   100.00 | NULL                                               |
	|  1 | SIMPLE      | business | NULL       | ALL  | PRIMARY       | NULL | NULL    | NULL |    2 |   100.00 | Using where; Using join buffer (Block Nested Loop) |
	+----+-------------+----------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
	2 rows in set, 1 warning (0.01 sec)
	
参数说明
	
	select_type:	   SELECT的类型 	
	table:             输出结果集的表
	partitions:        
	type:              表的连接类型
	possible_keys:     可能会使用到的索引
	key:               实际使用到的索引
	key_len:           索引字段的长度
	ref:               
	rows:              扫描的行数
	filtered:          
	extra:             执行情况的说明与描述

这里 explain 工具的使用，请自行查阅  

#### 通过show profile分析SQL资源占用 定位具体原因

查询profile配置情况
	
	mysql> show variables like "%profiling%";
	+------------------------+-------+
	| Variable_name          | Value |
	+------------------------+-------+
	| have_profiling         | YES   |
	| profiling              | OFF   |
	| profiling_history_size | 15    |
	+------------------------+-------+
	3 rows in set (0.01 sec)

打开profiling
	
	set profiling=on

执行SQL	

	mysql> select * from region left join business on region.business_id=business.business_id;

进行分析
	
	mysql> show profiles;
	+----------+------------+------------------------------------------------------------------------------------+
	| Query_ID | Duration   | Query                                                                              |
	+----------+------------+------------------------------------------------------------------------------------+
	|        1 | 0.05457050 | select * from region left join business on region.business_id=business.business_id |
	+----------+------------+------------------------------------------------------------------------------------+
	1 rows in set, 1 warning (0.00 sec)

show profiles 有许多参数 
	
	mysql> show profile cpu;
	+----------------------+----------+----------+------------+
	| Status               | Duration | CPU_user | CPU_system |
	+----------------------+----------+----------+------------+
	| starting             | 0.000111 | 0.000000 |   0.000000 |
	| checking permissions | 0.000019 | 0.000000 |   0.000000 |
	| checking permissions | 0.042538 | 0.000000 |   0.001000 |
	| Opening tables       | 0.011505 | 0.000000 |   0.000000 |
	| init                 | 0.000025 | 0.000000 |   0.000000 |
	| System lock          | 0.000020 | 0.000000 |   0.000000 |
	| optimizing           | 0.000019 | 0.000000 |   0.000000 |
	| statistics           | 0.000039 | 0.000000 |   0.000000 |
	| preparing            | 0.000062 | 0.000000 |   0.000000 |
	| executing            | 0.000008 | 0.000000 |   0.000000 |
	| Sending data         | 0.000121 | 0.000000 |   0.000000 |
	| end                  | 0.000009 | 0.000000 |   0.000000 |
	| query end            | 0.000030 | 0.000000 |   0.000000 |
	| closing tables       | 0.000018 | 0.000000 |   0.000000 |
	| freeing items        | 0.000031 | 0.000000 |   0.000000 |
	| cleaning up          | 0.000017 | 0.000000 |   0.000000 |
	+----------------------+----------+----------+------------+
	16 rows in set, 1 warning (0.00 sec)


#### 通过trace分析优化器如何选择执行计划

这个没实操过，等我掌握了再补充 (——_——)

### step.4 根据分析结果，确定问题并采取对应的优化措施

常见SQL层次优化技巧
	
	索引优化
		最常规的优化手段
	调整业务逻辑
		不要过于依赖一条SQL解决所有问题
	使用存储过程
		数据库分担一部分计算压力
	
数据库对象优化

	分库分表(垂直拆分、水平拆分)，提高访问效率
		将常用字段与非常用字段进行垂直拆分，将可归档的数据进行水平拆分
	使用中间表提高查询速度
	数据缓存策略
	数据冗余策略

## Part.2 应用方面

应用的优化是数据库优化的重要组成部分

### 减少MySQL连接的创建

客户端连接MySQL服务器，三次握手数据包就要在客户端与服务端之间至少往返七次，再加上连接变量的设置，比如字符集、是否自动提交事务等

	<?php
	
	for($i = 0; $i < 1 ;$i++)
	{
	    $con = mysqli_connect('172.16.125.253','root','xxxxxx','qq');
	}


使用tcpdump监测数据包

	[root@vagrant-centos65 bp]# tcpdump -i any port 3306 -S
	tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
	listening on any, link-type LINUX_SLL (Linux cooked), capture size 65535 bytes
	
	// 	创建连接
	17:34:41.422072 IP 172.16.125.253.57380 > 172.16.125.253.mysql: Flags [S], seq 2382269401, win 32792, options [mss 16396,sackOK,TS val 28893111 ecr 0,nop,wscale 6], length 0
	17:34:41.422086 IP 172.16.125.253.mysql > 172.16.125.253.57380: Flags [S.], seq 1999127578, ack 2382269402, win 32768, options [mss 16396,sackOK,TS val 28893111 ecr 28893111,nop,wscale 6], length 0
	17:34:41.422098 IP 172.16.125.253.57380 > 172.16.125.253.mysql: Flags [.], ack 1999127579, win 513, options [nop,nop,TS val 28893111 ecr 28893111], length 0
	
	// 用户认证
	17:34:41.422878 IP 172.16.125.253.mysql > 172.16.125.253.57380: Flags [P.], seq 1999127579:1999127657, ack 2382269402, win 512, options [nop,nop,TS val 28893112 ecr 28893111], length 78
	17:34:41.422901 IP 172.16.125.253.57380 > 172.16.125.253.mysql: Flags [.], ack 1999127657, win 513, options [nop,nop,TS val 28893112 ecr 28893112], length 0
	17:34:41.422977 IP 172.16.125.253.57380 > 172.16.125.253.mysql: Flags [P.], seq 2382269402:2382269511, ack 1999127657, win 513, options [nop,nop,TS val 28893112 ecr 28893112], length 109
	17:34:41.422983 IP 172.16.125.253.mysql > 172.16.125.253.57380: Flags [.], ack 2382269511, win 512, options [nop,nop,TS val 28893112 ecr 28893112], length 0
	17:34:41.423097 IP 172.16.125.253.mysql > 172.16.125.253.57380: Flags [P.], seq 1999127657:1999127668, ack 2382269511, win 512, options [nop,nop,TS val 28893112 ecr 28893112], length 11
	17:34:41.423252 IP 172.16.125.253.57380 > 172.16.125.253.mysql: Flags [P.], seq 2382269511:2382269516, ack 1999127668, win 513, options [nop,nop,TS val 28893112 ecr 28893112], length 5
	
	// 断开连接
	17:34:41.423312 IP 172.16.125.253.mysql > 172.16.125.253.57380: Flags [F.], seq 1999127668, ack 2382269516, win 512, options [nop,nop,TS val 28893112 ecr 28893112], length 0
	17:34:41.423398 IP 172.16.125.253.57380 > 172.16.125.253.mysql: Flags [F.], seq 2382269516, ack 1999127669, win 513, options [nop,nop,TS val 28893112 ecr 28893112], length 0
	17:34:41.423413 IP 172.16.125.253.mysql > 172.16.125.253.57380: Flags [.], ack 2382269517, win 512, options [nop,nop,TS val 28893112 ecr 28893112], length 0


当数据库的并发量变大时，连接的创建数直接影响数据库服务器的并发能力。那么，如何减少对MySQl的访问量呢

- 创建应用端连接池，实现对连接的复用
- 在应用端增加Cache层，实现对数据的复用	

## Part.3 MySQL服务架构

### 负载均衡 Load Balance

通过均衡算法，将固定负载量平均到多台服务器，以此减轻单体服务器的负载而达到优化的目的。负载均衡可以用在服务器的各个层面

    web服务器
    应用服务器
    数据服务器
