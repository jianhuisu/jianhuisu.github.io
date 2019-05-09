---
title : MySQL优化
categories : 
 - mysql 
tags :
	- MySQL
---

MySQL的优化主要分为三个方面

1. SQL语句优化
1. 应用层面
1. MySQL服务架构

本篇笔记主要从整体角度对SQL调优的步骤进行梳理，便于转化为自己的知识体系，因为篇幅限制，文中涉及到的重要的知识点会后续更新中补充

## Part.1 SQL语句优化

### step.1 对服务概况有基本了解

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


show [global|session] status like "Handler_read%";

	mysql> show global status like "Handler_read%";
	+-----------------------+-------+
	| Variable_name         | Value |
	+-----------------------+-------+
	| Handler_read_first    | 121   |
	| Handler_read_key      | 3833  |
	| Handler_read_last     | 0     |
	| Handler_read_next     | 5784  |
	| Handler_read_prev     | 0     |
	| Handler_read_rnd      | 604   |
	| Handler_read_rnd_next | 11067 |
	+-----------------------+-------+
	7 rows in set (0.00 sec)

`Handler_read_key` 值越高，代表索引的使用率越高
`Handler_read_rnd_next` 值越高 代表查询效率低，并且应该建立索引补救

备注：

- show [global|session] status    查看mysql运行状态参数
- show [global|session] variables 查看mysql服务配置

### step.2 收集慢查询SQL

#### 慢查询日志

查看慢查询日志是否开启

	mysql> show variables like "%slow%";
	+---------------------------+-------------------------------------------+
	| Variable_name             | Value                                     |
	+---------------------------+-------------------------------------------+
	| log_slow_admin_statements | OFF                                       |
	| log_slow_slave_statements | OFF                                       |
	| slow_launch_time          | 2                                         |
	| slow_query_log            | OFF                                        |
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
	
`slow_query_log` 代表慢查询是否开启 OFF:未开启 ON:开启

开启慢查询日志

慢查询日志的开启有两种方式

- 在线开启
- 通过配置文件开启

1 在线开启

	mysql> set global slow_query_log=1;
	1 row in set (0.01 sec)
	
	mysql> show variables like "%slow%";
	+---------------------------+-------------------------------------------+
	| Variable_name             | Value                                     |
	+---------------------------+-------------------------------------------+
	| log_slow_admin_statements | OFF                                       |
	| log_slow_slave_statements | OFF                                       |
	| slow_launch_time          | 2                                         |
	| slow_query_log            | OFF                                       |
	| slow_query_log_file       | /var/lib/mysql/vagrant-centos6.5-slow.log |
	+---------------------------+-------------------------------------------+
	5 rows in set (0.01 sec)

在线关闭
	
	mysql> set global slow_query_log=0;
	1 row in set (0.01 sec)	

个人感觉这种在线开启,关闭的方式非常实用。如果运营突然反馈感觉系统有点慢,开启慢查询日志阶段性监测是个不错的选择
	
2 通过配置文件开启 `vim /etc/my.cnf`


	[mysqld]
	...

	slow_query_log = ON
    slow_query_log_file = /var/lib/mysql/vagrant-centos6.5-slow.log
    long_query_time = 2

保存退出后需要重启`msyqld`服务。有时候重启服务的代价有点大,生产环境慎用。

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

下面是一个生产环境上抓的慢查询片段实例
	
	# Time: 2019-05-09T02:55:59.536527Z
	# User@Host: root[root] @ localhost [127.0.0.1]  Id: 58700
	# Query_time: 10.810361  Lock_time: 0.000121 Rows_sent: 1  Rows_examined: 393855
	SET timestamp=1557370559;
	SELECT ...FROM ...;
		
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

	mysql> show profile all;
	... // 这个更详细

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

具体使用方法查阅在线文档

	mysql> ? profile

#### 通过trace分析优化器如何选择执行计划

这个没实操过，等我掌握了再补充 (——_——)

#### show [full] processlist 实时监测mysql的连接

慢查询日志只会对已经执行完成SQL记录，但是如果现在MySQL很卡，如何实时查看呢,这就需要借助`show processlist`命令

	mysql> show full processlist;
	+----+-----------------+----------------------+-------+-------------+------+---------------------------------------------------------------+-----------------------+
	| Id | User            | Host                 | db    | Command     | Time | State                                                         | Info                  |
	+----+-----------------+----------------------+-------+-------------+------+---------------------------------------------------------------+-----------------------+
	|  4 | event_scheduler | localhost            | NULL  | Daemon      | 8314 | Waiting on empty queue                                        | NULL                  |
	|  9 | slave_account   | 172.16.125.217:45976 | NULL  | Binlog Dump | 8071 | Master has sent all binlog to slave; waiting for more updates | NULL                  |
	| 10 | root            | localhost            | NULL  | Query       |    0 | starting                                                      | show full processlist |
	| 11 | root            | 172.16.125.153:61557 | study | Sleep       |    0 |                                                               | NULL                  |
	+----+-----------------+----------------------+-------+-------------+------+---------------------------------------------------------------+-----------------------+
	4 rows in set (0.00 sec)

- Id 连接ID,常在手动断开连接时使用
- User  显示当前用户，如果不是root，这个命令就只显示你权限范围内的sql语句
- Host 显示连接的来源`ip`、`port`,常用来追踪用户
- db 连接访问的哪个db
- Command 当前连接负责任务的简单描述，例如 sleep query connect、Binlog Dump 等 
- Time 会话持续时长
- State 当前连接执行任务内容的详细描述
- Info 具体信息,比如正在执行的sql语句

Tips:Command列中如果产生大量sleep连接,最大的可能是mysql的调用方在获取结果集完成后,没有及时释放连接,转而进行其它的操作,导致调用方的进程不结束,
mysql连接一直得不到释放。

如果发现某个恶意连接，可以直接杀掉

	mysql> show full processlist;
	+----+-----------------+----------------------+------+---------+------+------------------------+-----------------------+
	| Id | User            | Host                 | db   | Command | Time | State                  | Info                  |
	+----+-----------------+----------------------+------+---------+------+------------------------+-----------------------+
	|  4 | event_scheduler | localhost            | NULL | Daemon  | 6092 | Waiting on empty queue | NULL                  |
	|  8 | root            | localhost            | qq   | Query   |    0 | starting               | show full processlist |
	| 19 | root            | 172.16.125.191:51591 | qq   | Query   |    4 | User sleep             | select sleep(10000)   |
	+----+-----------------+----------------------+------+---------+------+------------------------+-----------------------+
	3 rows in set (0.00 sec)

	mysql> kill query 19;
	Query OK, 0 rows affected (0.00 sec)

备注：

- MySQL是单进程多线程应用。主进程挂了,整个服务也就挂了。
- Oracle在linux上默认是多进程，多进程的好处是一个进程崩溃不会影响另外的进程

### step.4 根据分析结果，确定问题并采取对应的优化措施

常见SQL层次优化技巧

	索引优化
		最常规的优化手段
	调整业务逻辑
		不要过于依赖一条SQL解决所有问题,越复杂的SQL越不稳定
	使用存储过程
		数据库分担一部分计算压力

数据库对象优化

#### 分库分表(垂直拆分、水平拆分)，提高访问效率

垂直拆分：根据数据的活跃度进行拆分，降低IO。例如博客的标题 创建时间等 变化频率低 使用MyISAM表引擎 为冷数据。博客的浏览量 、博客的点赞量实时变化为热数据 使用Innodb方便行数据更新。

水平拆分：根据具体的业务需求结合一定规则进行，分享一个栗子 [http://www.jb51.net/article/29771.htm](http://www.jb51.net/article/29771.htm "分表")

一条比较实用的SQL：

	create table new_user like user;
	RENAME TABLE members TO members_bak,members_tmp TO members;

####	使用中间表提高查询速度

	根据业务情况,比如在多表联查时手动创建临时表,尽快释放表资源。

####	数据缓存策略

####	数据冗余策略

允许部分冗余减少连接查询

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
- 在应用端增加Cache层，实现对数据的复用(离线分析、流式计算)
- 读写分离

## Part.3 MySQL服务架构

### 负载均衡 Load Balance

通过均衡算法，将负载量分摊到多台服务器，以此减轻单体服务器的负载而达到优化的目的。负载均衡可以用在服务器的各个层面

    web服务器
    应用服务器
    数据服务器(分布式数据库、数据库集群)

#### MySQL Cluster

至少部署两台MySQL服务器构成一个小的集群，主要有2个目的：

- 高可用性：在主机挂掉后，只产生单点故障 自动故障转移，使前端服务对用户无影响。
- 读写分离：将主库读压力分流到从库上。 可在客户端组件上实现负载均衡，根据不同服务器的运行情况，分担不同比例的读请求压力。

集群的节点主要分为三个层次

- 管理节点   不直接参与应用的访问 ，主要监测 集群的运行状态 、数据同步
- SQL节点    负责资源的调度
- 数据节点   负责实际资源的存储

备注：

- 只有Innodb支持分布式事务
- 全局事务需要应用实现

#### 分布式数据库

例如TiDB

### 优化数据库对象

定期检查、优化表  （optimize table ，check table）
充分利用列有默认值
表的字段尽量不使用自增长变量，在高并发情况下自增对性能有较大影响
将表分盘存储，平均磁盘IO


## Part.4 操作系统

提高硬件条件带来的硬件成本在一定场景下成本要远小于人工成本

- 硬 RAID  硬磁盘阵列  分布式存储数据，保障数据的准确性
- 软 RAID  软磁盘阵列  通过算法模拟磁盘阵列的存储方式

## 总结

当然，还有很多优化的方向，比如内存优化等 但是目前我还没有掌握，等实操后，我再总结了。


## 参考资料

深入浅出mysql

https://blog.csdn.net/dhfzhishi/article/details/81263084

[https://www.cnblogs.com/hhandbibi/p/7118740.html](https://www.cnblogs.com/hhandbibi/p/7118740.html "https://www.cnblogs.com/hhandbibi/p/7118740.html")