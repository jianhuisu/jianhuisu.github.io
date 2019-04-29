---
title : MySQL支持的数据类型
categories : 
 - mysql 
tags :
	- MySQL
---

想要熟练的创建数据表，就需要打好牢固的mysql基础


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

### 时间类型


| 时间类型        |  字节            |   格式  |
| ------------- |:-------------:   | -----:|
| DATE          | 4                | 2018-12-20 |
| DATETIME      | 8                |   2018-12-20 10:54:58 |
| TIMESTAMP     | 4                |    2018-12-20 10:54:58 |
| TIME          | 3                |    10:54:58 |
| YEAR          | 1                |    2018 |

创建时间类型表

	mysql> create table tmp_table_date(id1 DATE,id2 DATETIME,id3 TIMESTAMP,id4 TIME,id5 YEAR);
	Query OK, 0 rows affected (0.15 sec)

	mysql> desc tmp_table_date;
	+-------+-----------+------+-----+---------+-------+
	| Field | Type      | Null | Key | Default | Extra |
	+-------+-----------+------+-----+---------+-------+
	| id1   | date      | YES  |     | NULL    |       |
	| id2   | datetime  | YES  |     | NULL    |       |
	| id3   | timestamp | YES  |     | NULL    |       |
	| id4   | time      | YES  |     | NULL    |       |
	| id5   | year(4)   | YES  |     | NULL    |       |
	+-------+-----------+------+-----+---------+-------+
	5 rows in set (0.01 sec)


将当前时间插入

	mysql> insert into tmp_table_date(id1,id2,id3,id4,id5) values(now(),now(),now(),now(),now());
	Query OK, 1 row affected, 1 warning (0.12 sec)

	mysql> select * from tmp_table_date;
	+------------+---------------------+---------------------+----------+------+
	| id1        | id2                 | id3                 | id4      | id5  |
	+------------+---------------------+---------------------+----------+------+
	| 2018-12-20 | 2018-12-20 10:54:58 | 2018-12-20 10:54:58 | 10:54:58 | 2018 |
	+------------+---------------------+---------------------+----------+------+
	1 row in set (0.00 sec)

这里使用了三个常见时间函数，函数返回值随系统时间改变


	mysql> SELECT NOW(),CURDATE(),CURTIME();
	+---------------------+------------+-----------+
	| NOW()               | CURDATE()  | CURTIME() |
	+---------------------+------------+-----------+
	| 2018-12-20 11:00:35 | 2018-12-20 | 11:00:35  |
	+---------------------+------------+-----------+
	1 row in set (0.00 sec)

	mysql> SELECT NOW(),CURDATE(),CURTIME();
	+---------------------+------------+-----------+
	| NOW()               | CURDATE()  | CURTIME() |
	+---------------------+------------+-----------+
	| 2018-12-20 11:00:38 | 2018-12-20 | 11:00:38  |
	+---------------------+------------+-----------+
	1 row in set (0.00 sec)


`YEAR` 类型不会自动截取，插入错误

	mysql> insert into tmp_table_date(id1,id2,id3,id4,id5) values('2018-12-20 11:00:38','2018-12-20 11:00:38','2018-12-20 11:00:38','2018-12-20 11:00:38','2018-12-20 11:00:38');
	ERROR 1265 (01000): Data truncated for column 'id5' at row 1
	mysql> select * from tmp_table_date;
	+------------+---------------------+---------------------+----------+------+
	| id1        | id2                 | id3                 | id4      | id5  |
	+------------+---------------------+---------------------+----------+------+
	| 2018-12-20 | 2018-12-20 10:54:58 | 2018-12-20 10:54:58 | 10:54:58 | 2018 |
	| 2018-12-20 | 2018-12-20 11:05:35 | 2018-12-20 11:05:35 | 11:05:35 | 2018 |
	+------------+---------------------+---------------------+----------+------+
	2 rows in set (0.00 sec)

`DATE`,`TIME` 类型虽然插入成功，但是系统发出警告：

	mysql> insert into tmp_table_date(id1,id2,id3,id4,id5) values('2018-12-20 11:00:38','2018-12-20 11:00:38','2018-12-20 11:00:38','2018-12-20 11:00:38','2018');
	Query OK, 1 row affected, 2 warnings (0.09 sec)

	mysql> show warnings;
	+-------+------+-----------------------------------------------------------------------+
	| Level | Code | Message                                                               |
	+-------+------+-----------------------------------------------------------------------+
	| Note  | 1292 | Incorrect date value: '2018-12-20 11:00:38' for column 'id1' at row 1 |
	| Note  | 1292 | Incorrect time value: '2018-12-20 11:00:38' for column 'id4' at row 1 |
	+-------+------+-----------------------------------------------------------------------+
	2 rows in set (0.00 sec)

所以最正确的插入格式为

	mysql> insert into tmp_table_date(id1,id2,id3,id4,id5) values('2018-12-20','2018-12-20 11:00:38','2018-12-20 11:00:38','11:00:38','2018');
	Query OK, 1 row affected (0.08 sec)

	mysql> select * from tmp_table_date;
	+------------+---------------------+---------------------+----------+------+
	| id1        | id2                 | id3                 | id4      | id5  |
	+------------+---------------------+---------------------+----------+------+
	| 2018-12-20 | 2018-12-20 10:54:58 | 2018-12-20 10:54:58 | 10:54:58 | 2018 |
	| 2018-12-20 | 2018-12-20 11:05:35 | 2018-12-20 11:05:35 | 11:05:35 | 2018 |
	| 2018-12-20 | 2018-12-20 11:00:38 | 2018-12-20 11:00:38 | 11:00:38 | 2018 |
	+------------+---------------------+---------------------+----------+------+
	3 rows in set (0.00 sec)

备注：timestamp的显示会根据时区变化而变化，timestamp支持的时间范围较小，不适合存储跨度久远的时间


### 字符串

#### char 与 varchar 的区别

1 存储方式不同，char字段声明时固定存储大小，varchar存储时根据值大小存储

2 char字段存储时会忽略末尾的空格，varchar保留

	mysql> create table tmp_table_string(name1 char(4),name2 varchar(4));
	Query OK, 0 rows affected (0.08 sec)

	mysql> desc tmp_table_string;
	+-------+------------+------+-----+---------+-------+
	| Field | Type       | Null | Key | Default | Extra |
	+-------+------------+------+-----+---------+-------+
	| name1 | char(4)    | YES  |     | NULL    |       |
	| name2 | varchar(4) | YES  |     | NULL    |       |
	+-------+------------+------+-----+---------+-------+
	2 rows in set (0.00 sec)

	mysql> insert into tmp_table_string(name1,name2) values("s  ","s   ");
	Query OK, 1 row affected (0.24 sec)

	mysql> select * from tmp_table_string;
	+-------+-------+
	| name1 | name2 |
	+-------+-------+
	| s     | s     |
	+-------+-------+
	1 row in set (0.00 sec)

name1末尾的空格被删除，name2末尾的空格被保留

	mysql> select length(name1),length(name2) from tmp_table_string;
	+---------------+---------------+
	| length(name1) | length(name2) |
	+---------------+---------------+
	|             1 |             4 |
	+---------------+---------------+
	1 row in set (0.00 sec)

注意，忽略的是字符串末尾空格，而非字符串头

	mysql> insert into tmp_table_string(name1,name2) values(" s ","s   ");
	Query OK, 1 row affected (0.02 sec)

	mysql> select length(name1),length(name2) from tmp_table_string;
	+---------------+---------------+
	| length(name1) | length(name2) |
	+---------------+---------------+
	|             1 |             4 |
	|             2 |             4 |
	+---------------+---------------+
	2 rows in set (0.00 sec)

	mysql> select concat(name1,'_'),concat(name2,'_') from tmp_table_string;
	+-------------------+-------------------+
	| concat(name1,'_') | concat(name2,'_') |
	+-------------------+-------------------+
	| s_                | s   _             |
	|  s_               | s   _             |
	+-------------------+-------------------+
	2 rows in set (0.00 sec)

char(10) 中的 10代表什么? 10个字节还是10个字符？

	mysql> create table char_leng(name1 char(10),name2 varchar(5))charset=utf8;
	Query OK, 0 rows affected (0.06 sec)

插入11个字母到`name1`字段，失败

	mysql> insert into char_leng(name1,name2) values('ssssssssssd','sssssd');
	ERROR 1406 (22001): Data too long for column 'name1' at row 1

	mysql> insert into char_leng(name1,name2) values('ssssssssssd','sssssd');
	ERROR 1406 (22001): Data too long for column 'name1' at row 1

插入11个汉字到`name1`字段，成功

	mysql> insert into char_leng(name1,name2) values('一二三四五六七八九','sssss');
	Query OK, 1 row affected (0.00 sec)

插入1个汉字

	mysql> insert into char_leng(name1,name2) values('一','sssss');
	Query OK, 1 row affected (0.01 sec)

观察存储数据内容实际长度

	mysql> select length(name1),name1,name2 from char_leng;
	+---------------+-----------------------------+-------+
	| length(name1) | name1                       | name2 |
	+---------------+-----------------------------+-------+
	|            27 | 一二三四五六七八九          | sssss |
	|             3 | 一                          | sssss |
	+---------------+-----------------------------+-------+
	2 rows in set (0.00 sec)

	mysql> select version();
	+-----------+
	| version() |
	+-----------+
	| 5.7.23    |
	+-----------+
	1 row in set (0.00 sec)


由此可知 `char(10)` 中数字10代表的是**最大存储字符的个数**(汉字、数字、字母、特殊字符均计为一个字符呦)，分配字段空间时根据table的 (`字符集编码对应字节数 * 最大存储字符个数`) 分配定长空间。同理 `varchar(10)` 代表的意思是,最多为该字段分配(`字符集编码对应字节数 * 最大存储字符个数`)空间，但是如果插入值按表的字符集编码换算为字节数后未超过最大占用空间，那么以插入值实际大小进行存储。


上面简单的介绍了mysql的各种数据类型，包括基本用途、物理存储、表示范围，这样在面对具体的应用时，可以根据应用的特点，争取在满足应用需求的基础上，用较小的存储代价换取较高的数据性能。

#### BINARY 与 VARBINARY

#### ENUM 与 SET

	mysql> create table tmp_table_enum( id int(11),name enum('M','F') );
	Query OK, 0 rows affected (0.21 sec)

	mysql> desc tmp_table_enum;
	+-------+---------------+------+-----+---------+-------+
	| Field | Type          | Null | Key | Default | Extra |
	+-------+---------------+------+-----+---------+-------+
	| id    | int(11)       | YES  |     | NULL    |       |
	| name  | enum('M','F') | YES  |     | NULL    |       |
	+-------+---------------+------+-----+---------+-------+
	2 rows in set (0.01 sec)

使用位置索引或者值本身均可实现插入

	mysql> insert into tmp_table_enum(id,name) values(1,1);
	Query OK, 1 row affected (0.19 sec)

	mysql> select * from tmp_table_enum;
	+------+------+
	| id   | name |
	+------+------+
	|    1 | M    |
	+------+------+
	1 row in set (0.00 sec)

	mysql> insert into tmp_table_enum(id,name) values(1,2);
	Query OK, 1 row affected (0.09 sec)

	mysql> select * from tmp_table_enum;
	+------+------+
	| id   | name |
	+------+------+
	|    1 | M    |
	|    1 | F    |
	+------+------+
	2 rows in set (0.00 sec)


	mysql> select * from tmp_table_enum where name='F';
	+------+------+
	| id   | name |
	+------+------+
	|    1 | F    |
	+------+------+
	1 row in set (0.00 sec)


## 小技巧

使用 procedure analyse() 现有字段进行分析

	mysql> create table analyse_table(id int() auto_increment primary key,name char(50),pid int(11))engine=MyISAM charset=utf8;
	ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ') auto_increment primary key,name char(50),pid int(11))engine=MyISAM charset=utf' at line 1
	mysql> create table analyse_table(id int(11) auto_increment primary key,name char(50),pid int(11))engine=MyISAM charset=utf8;
	Query OK, 0 rows affected (0.00 sec)

	mysql> insert into analyse_table(name,pid) values('su',1),('zhao',1),('bao',1),('fu',1),('zhang',1);
	Query OK, 5 rows affected (0.01 sec)
	Records: 5  Duplicates: 0  Warnings: 0

	mysql> select * from analyse_table procedure analyse(1);
	+---------------------------+-----------+-----------+------------+------------+------------------+-------+-------------------------+--------+-----------------------------------------------+
	| Field_name                | Min_value | Max_value | Min_length | Max_length | Empties_or_zeros | Nulls | Avg_value_or_avg_length | Std    | Optimal_fieldtype                             |
	+---------------------------+-----------+-----------+------------+------------+------------------+-------+-------------------------+--------+-----------------------------------------------+
	| test_1.analyse_table.id   | 1         | 5         |          1 |          1 |                0 |     0 | 3.0000                  | 1.4142 | TINYINT(1) UNSIGNED NOT NULL                  |
	| test_1.analyse_table.name | bao       | zhao      |          2 |          5 |                0 |     0 | 3.2000                  | NULL   | ENUM('bao','fu','su','zhang','zhao') NOT NULL |
	| test_1.analyse_table.pid  | 1         | 1         |          1 |          1 |                0 |     0 | 1.0000                  | 0.0000 | TINYINT(1) UNSIGNED NOT NULL                  |
	+---------------------------+-----------+-----------+------------+------------+------------------+-------+-------------------------+--------+-----------------------------------------------+
	3 rows in set, 1 warning (0.00 sec)

怎么样，是不是很实用，不过这个工具在MySQL8.0被移除了

	mysql> ? procedure analyse;
	Name: 'PROCEDURE ANALYSE'
	Description:
	Syntax:
	ANALYSE([max_elements[,max_memory]])

	*Note*:

	PROCEDURE ANALYSE() is deprecated as of MySQL 5.7.18, and is removed in
	MySQL 8.0.

	ANALYSE() examines the result from a query and returns an analysis of
	the results that suggests optimal data types for each column that may
	help reduce table sizes. To obtain this analysis, append PROCEDURE
	ANALYSE to the end of a SELECT statement:

	SELECT ... FROM ... WHERE ... PROCEDURE ANALYSE([max_elements,[max_memory]])




### TODO

我有两个疑问

1. 在我们日常开发过程中 我们一般使用`int`类型来存储时间戳，在检索数据时，对int排序与对datetime排序哪个效率高
1. 使用datetime存储时间，现实世界时间超过数据库允许存储的最大值应该如何处理
