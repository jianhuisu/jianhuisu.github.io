---
title : mysql主从复制搭建
categories : 
 - mysql 
tags :
	- Mysql
---

首先描述一下主从同步的原理:

1. mysql是单进程多线程的应用,主库需要将自身数据的变动写入到binlog,供从库拉取。
1. master服务启动一个线程,负责记录数据变动至binlog
1. slave服务启动一个线程,负责拉取主库的binlog

(同步的指导思想才是精华,下面的具体操作步骤只不过是思想的表象,会很快的更迭掉)

## 主从同步搭建

主库:

	IP:172.16.124.67
	OS:centos6.5
	mysql:8.0.11

从库:

	IP:172.16.124.71
	OS:centos7.4
	mysql:8.0.15


### Step.1 master开启binlog

vim /etc/my.cnf

	[mysqld]

	...

	# master
	server-id=1
	log-bin=mysql-bin
	binlog-do-db=study
	binlog-ignore-db=mysql

保存退出,重启服务

### Step.2 设置slave mysql 使用账号


	mysql> create user 'slave_account1'@'%' identified by 'yigefuzaqiehenchangdemima_A12b3C4d';
	Query OK, 0 rows affected (0.10 sec)

	mysql> GRANT REPLICATION SLAVE ON *.* TO 'slave_account1'@'%';
	Query OK, 0 rows affected (0.05 sec)

	mysql> flush privileges;
	Query OK, 0 rows affected (0.01 sec)


不同版本的mysql设置命令不同,但大体思路不变。
获取`File`、`Position`参数供slave配置时使用。

	mysql> show master status;
	+------------------+----------+--------------+------------------+-------------------+
	| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
	+------------------+----------+--------------+------------------+-------------------+
	| mysql-bin.000005 |     1187 | study        | mysql            |                   |
	+------------------+----------+--------------+------------------+-------------------+
	1 row in set (0.00 sec)


### Step.3 从库配置

vim /etc/my.cnf

	[mysqld]

	...

	# slave
	server-id=2
	...

保存退出,重启服务。

登入mysql控制台,向master注册slave_host

	mysql> CHANGE MASTER TO MASTER_HOST='172.16.124.67',MASTER_USER='slave_account1',MASTER_PASSWORD='yigefuzaqiehenchangdemima_A12b3C4d',MASTER_LOG_FILE='mysql-bin.000005',MASTER_LOG_POS=1187;
	Query OK, 0 rows affected (0.01 sec)

MASTER_LOG_FILE、MASTER_LOG_POS 这两个参数的值需要在master控制台中执行`show master status`获取

### Step.4 初始数据设置

如果对master的配置文件进行思考,很容易发现主从同步时只会同步库内操作,并且重做命令只会从当前时间点开始执行,历史数据需要我们手动同步

将主库中的数据打包发送到从库

	mysql>flush tables with read lock;
	...
	[root@vagrant-centos65 ~]$> mysqldump -uroot -paaaaaa study > study.sql
	[root@vagrant-centos65 ~]$> scp study.sql root@172.16.124.71:/root/
	...
	mysql>unlock tables;

从库接收数据

	mysql -uroot -padmin study<study.sql


### Step.5 从库启动同步线程

	mysql> start slave;
	Query OK, 0 rows affected (0.01 sec)

	mysql> show slave status\G
	*************************** 1. row ***************************
				   Slave_IO_State: Waiting for master to send event
					  Master_Host: 172.16.124.67
					  Master_User: slave_account
					  Master_Port: 3306
					Connect_Retry: 60
				  Master_Log_File: mysql-bin.000005
			  Read_Master_Log_Pos: 1187
				   Relay_Log_File: localhost-relay-bin.000004
					Relay_Log_Pos: 1401
			Relay_Master_Log_File: mysql-bin.000005
				 Slave_IO_Running: Yes
				Slave_SQL_Running: Yes
				...

`Slave_IO_Running`、`Slave_SQL_Running`两个值均为yes代表同步健康

Slave_IO_Running：连接到主库，并读取主库的日志到本地，生成本地日志文件
Slave_SQL_Running:读取本地日志文件，并执行日志里的SQL命令。


## 主从同步监测

搭建成功后,需要对数据的同步状态进行监测,谁知道投入使用后会出现什么问题

### 监测主库同步线程是否健康

	mysql> show full processlist\G
	...
	*************************** 3. row ***************************
		 Id: 11
	   User: slave_account
	   Host: 172.16.124.71:59362
		 db: NULL
	Command: Binlog Dump
	   Time: 511
	  State: Master has sent all binlog to slave; waiting for more updates
	   Info: NULL

	mysql> show master status\G

### 监测从库同步线程是否健康

	mysql> show full processlist\G
	...
	*************************** 4. row ***************************
		 Id: 21
	   User: system user
	   Host:
		 db: NULL
	Command: Query
	   Time: 661
	  State: Slave has read all relay log; waiting for more updates
	   Info: NULL
	4 rows in set (0.00 sec)

	mysql> show slave status\G

### 查看在master注册的slave

	mysql> show slave hosts;
	+-----------+------+------+-----------+--------------------------------------+
	| Server_id | Host | Port | Master_id | Slave_UUID                           |
	+-----------+------+------+-----------+--------------------------------------+
	|         2 |      | 3306 |         1 | d81cc1d0-6686-11e9-b887-525400cae48b |
	+-----------+------+------+-----------+--------------------------------------+
	1 row in set (0.00 sec)

## 常见错误解决

### "Slave_SQL_Running:No"

重置同步的偏移量:

	1. 从库  stop slave 手动同步数据
	2. 主库  mysql> show master status;
	3. 从库  获取 主库的偏移量,从库重置
		CHANGE MASTER TO MASTER_HOST='172.16.124.67',MASTER_USER='slave_account',MASTER_PASSWORD='aaaaaa',MASTER_LOG_FILE='mysql-bin.000004',MASTER_LOG_POS=951;
	4. start slave

### Slave_IO_Running:NO

	从库的Slave未启动

## 小结

slave服务器停机一段时间,当slave再次开机时,这段时间内的差异数据会在开机后进行同步。但是,如果slave停机时间过长,导致master的binlog已经过期,会产生数据断层。另外为了保险起见,在重启slave之后,要检查一下slave线程是否执行正常。
master数据库服务器停机后,slave处于等待同步状态,但是不影响查询,


## 参考资料

搭建 https://www.cnblogs.com/zhoujie/p/mysql1.html

同步错误处理 https://www.cnblogs.com/rongfengliang/p/5727087.html

binlog介绍(强推) https://www.cnblogs.com/rjzheng/p/9721765.html




