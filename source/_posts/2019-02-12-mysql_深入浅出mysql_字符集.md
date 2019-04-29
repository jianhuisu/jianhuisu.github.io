---
title : MySQL的字符集
categories : 
 - mysql 
tags :
	- MySQL
---

## 字符集的概念

为了使计算机可以处理文字信息，使用二进制对文字进行编码、存储、表示。

ASCII   美国标准化组织  单字节
ISO      国际标准化组织
Unicode 双字节  节省内存、节省时间
BMP  basic multi-lingual plane  基本字面
UCS-4  ISO-10646   （4 字节）

group   第一个字节
plane   第二个字节
row     第三个字节
ceil    第四个字节

UTF-8    变长  1-4 字节编码  汉字3字节 英文 一字节
UTF-16  变长

## 数据库中有关字符集的设置

MySQL的字符集包括 字符集与校对规则两个概念
字符集：决定数据存储的方式   CHARACTER
校对规则:决定字符串比较的方式      COLLATION

查看MySQL支持的字符集

	show character set
	mysql> show variables like 'character_set_server';
	+----------------------+-------+
	| Variable_name        | Value |
	+----------------------+-------+
	| character_set_server | utf8  |
	+----------------------+-------+
	1 row in set (0.00 sec)

	mysql> show variables like 'collcation_server';
	Empty set (0.00 sec)

	mysql> show variables like 'character_set_client';
	+----------------------+-------+
	| Variable_name        | Value |
	+----------------------+-------+
	| character_set_client | utf8  |
	+----------------------+-------+
	1 row in set (0.00 sec)

	mysql> show variables like 'collcation_client';
	Empty set (0.00 sec)

	mysql> show variables like 'character_set_database';
	+------------------------+-------+
	| Variable_name          | Value |
	+------------------------+-------+
	| character_set_database | utf8  |
	+------------------------+-------+
	1 row in set (0.01 sec)



MySQL的字符集、校对规则设置分为四个级别

### 服务器的字符集设置

	my.cnf

	[mysqld]
	  collation-server = utf8_unicode_ci
	  character-set-server=utf8

	启动mysql进程时设置

	mysqld --default-character-set=gbk
	编译时设置
	./configure --with-charset=gbk

如果没有特别指定服务器字符集，默认使用 latin1 作为服务器的字符集

查询当前服务器的字符集设置

	show variables like 'character_set_server';

查询当前服务器的校对规则设置

	show variables like 'collation_server';

### 数据库的字符集设置

- 如果指定了字符集与校对规则，那么使用指定的字符集与校对规则
- 	如果只指定了字符集，没有指定校对规则，那么使用指定字符集与指定字符集默认的校对规则
- 	如果没有指定字符集与校对规则，那么使用服务器的字符集与校对规则

查询当前数据库的字符集设置

	show variables like 'character_set_database';

查询当前数据库的字符集的校对规则

	show variables like 'collation_database';


### 数据表的字符集设置

- 	如果指定了字符集与校对规则，那么使用指定的字符集与校对规则
- 	如果只指定了字符集，没有指定校对规则，那么使用指定字符集与指定字符集默认的校对规则
- 	如果没有指定字符集与校对规则，那么使用数据库的字符集与校对规则

如何查看数据表的字符集

	mysql> show table status like 'tgbk'\G;
	*************************** 1. row ***************************
	           Name: tgbk
	         Engine: InnoDB
	        Version: 10
	     Row_format: Dynamic
	           Rows: 0
	 Avg_row_length: 0
	    Data_length: 16384
	Max_data_length: 0
	   Index_length: 0
	      Data_free: 0
	 Auto_increment: NULL
	    Create_time: 2019-02-20 11:07:25
	    Update_time: NULL
	     Check_time: NULL
	      Collation: gbk_chinese_ci   // 这里校对规则的前缀就是该表的字符集
	       Checksum: NULL
	 Create_options:
	        Comment:
	1 row in set (0.00 sec)



### 字段的字符集设置

	一般很少遇到，这里不做记录

备注：建库建表时显示指定字符集与校对规则

## 如何选择合适的字符集

选择合适的字符集对于以后系统的性能、推广、移植 都有重要的影响

- 1 变更数据库的字符集时，要兼容原有的字符集，即新字符集要为原字符集的超集，这样可以避免因为字符集不兼容的问题造成数据损失、乱码
- 2 数据库运算量大，性能要求较高 ，尽量使用定长字符集，定长字符集的运算较变长稍快
- 3 满足应用面向用户群体的语言需求 （如果需要面向国际，需要使用 unicode ）
- 4 优先选择 可以保证所有客户端字符集一致的字符集，避免字符集转化带来的性能开销与数据损失
- 5 如果绝大部分内容为汉字，可以考虑使用GBK代替UTF-8  ，双字节代替三字节 。可以降低磁盘IO，数据库cache、网络传输时间  从而提高性能。

## 连接的字符集

每次客户端与MySQL服务器进行交互，都需要声明 character_set_client、character_set_connection、character_set_result
这三个参数保持一致，才可以保障数据正确的录入读出。

	mysql> show variables like 'character_set_client';
	+----------------------+-------+
	| Variable_name        | Value |
	+----------------------+-------+
	| character_set_client | utf8  |
	+----------------------+-------+
	1 row in set (0.05 sec)

	mysql> show variables like 'character_set_connection';
	+--------------------------+-------+
	| Variable_name            | Value |
	+--------------------------+-------+
	| character_set_connection | utf8  |
	+--------------------------+-------+
	1 row in set (0.00 sec)

	mysql> show variables like 'character_set_result';
	Empty set (0.01 sec)



## 字符集的修改

**更改字符集时，已存在的数据不会自动更新为新的字符集,需要对已存在的数据进行转码
**
