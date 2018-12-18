---
title ： MySQL支持的数据类型
---

想要熟练的创建数据表，就需要牢固的掌握数据类型的取值范围，超出范围时该如何处理


### 整数 int 

在数据库中，一个int类型数字占用4个字节，即4*8=32位，那么一个unsigned int类型字段所能表示的最大取值区间为 `0~2^32-1`  
	
	mysql> ? int
	Name: 'INT'
	Description:
	INT[(M)] [UNSIGNED] [ZEROFILL]
	
	A normal-size integer. The signed range is -2147483648 to 2147483647.
	The unsigned range is 0 to 4294967295.

那么，理论上我插入`2147483648`到有符号的int型字段中是超出范围的,创建测试使用数据表：

	mysql> show create table data_category\G
	*************************** 1. row ***************************
	       Table: data_category
	Create Table: CREATE TABLE `data_category` (
	  `id1` int(4) DEFAULT NULL,
	  `id2` int(11) DEFAULT NULL
	) ENGINE=InnoDB DEFAULT CHARSET=utf8


	mysql> insert into data_category(id1,id2) values(2147483647,4);
	Query OK, 1 row affected (0.07 sec)
	
	mysql> insert into data_category(id1,id2) values(2147483648,4);
	ERROR 1264 (22003): Out of range value for column 'id1' at row 1

为避免发生`Out of range`错误，需要在表设计阶段选择足够大的数据类型来容纳可能的字段值。

### 小数

浮点数

	float(M,N)
	double(M,N)
定点数

	decimal(M,N) 实际上以字符串的方式存储

float(M,N)  

- M 称为精度，M为有效数字个数
- N 称为标度，N为小数点后有效数字个数

如果浮点数设置了精度与标度，则会自动将四舍五入的结果插入，如果没有设置精度与标度，则会按照实际精度值显示。系统不会报错。
定点数如果不写精度与标度，默认decimal(10,0)，如果超过精度和标度，系统会报错

todo 这里定点数的截取有问题

	mysql> desc decimal_number;
	+-------+--------------+------+-----+---------+-------+
	| Field | Type         | Null | Key | Default | Extra |
	+-------+--------------+------+-----+---------+-------+
	| id1   | float(5,2)   | YES  |     | NULL    |       |
	| id2   | double(5,2)  | YES  |     | NULL    |       |
	| id3   | decimal(5,2) | YES  |     | NULL    |       |
	+-------+--------------+------+-----+---------+-------+
	3 rows in set (0.00 sec)

进行插入，按照逻辑`id3`应该截取为`1.23`而不是`1.24`
	
	mysql> insert into decimal_number(id1,id2,id3) values(1.235,1.235,1.235);
	Query OK, 1 row affected, 1 warning (0.09 sec)
	
	mysql> select * from decimal_number;
	+------+------+------+
	| id1  | id2  | id3  |
	+------+------+------+
	| 1.24 | 1.24 | 1.24 |
	+------+------+------+
	1 row in set (0.00 sec)

	mysql> insert into decimal_number(id1,id2,id3) values(111.235,111.235,111.235);
	Query OK, 1 row affected, 1 warning (0.12 sec)
	
	mysql> select * from decimal_number;
	+--------+--------+--------+
	| id1    | id2    | id3    |
	+--------+--------+--------+
	|   1.24 |   1.24 |   1.24 |
	| 111.23 | 111.23 | 111.24 |
	+--------+--------+--------+
	2 rows in set (0.00 sec)
	
这里应该跟SQL_MODE有关，等待后续完善



	