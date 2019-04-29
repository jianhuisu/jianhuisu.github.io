---
title : 常见实用SQL优化技巧
categories : 
 - mysql 
tags :
	- MySQL
---

知识点有点散 ————__————

## Point.1 定期检查表，分析表

	check table table_name
	repair table table_name
	optimize table table_name
	select * from table_name procedure analyse(1)


## Point.2 批量导入数据的优化

未实操，待补充

## Point.3 插入数据的优化

- 从统一客户端插入大量数据时，使用批量插入替代单条插入
- 当从一个文件装载一个表时，使用`LOAD DATA INFILE`
- 从不同客户端进行插入时，使用`INSERT DELAYED` 先入内存队列，而后同步到磁盘、`LOW_PRIORITY`则相反，等待其它用户对表读完以后进行插入（默认`insert`操作的优先级是高于`select`的优先级）
- 将索引文件与数据文件分开存储（MyISAM存储引擎表），建表时指定存储目录


## Point.4 优化 GROUP BY

复习GROUP BY的语法

	SELECT [field1,field2,……fieldn] fun_name
	FROM tablename
	[WHERE where_contition]
	[GROUP BY field1,field2,……fieldn
	[WITH ROLLUP]]
	[HAVING where_contition]

对其参数进行以下说明：

- fun_name 表示要做的聚合操作，也就是聚合函数，常用的有 sum（求和）、count(*)（记
 	录数）、max（最大值）、min（最小值）。
- GROUP BY 关键字表示要进行分类聚合的字段，比如要按照部门分类统计员工数量，部门
 	就应该写在 group by 后面。
- WITH ROLLUP 是可选语法，表明是否对分类聚合后的结果进行再汇总。
- HAVING 关键字表示对分类后的结果再进行条件的过滤

使用group by时默认排序

	mysql> explain select * from table_group group by name;
	+----+-------------+-------------+------+---------------+------+---------+------+------+---------------------------------+
	| id | select_type | table       | type | possible_keys | key  | key_len | ref  | rows | Extra                           |
	+----+-------------+-------------+------+---------------+------+---------+------+------+---------------------------------+
	|  1 | SIMPLE      | table_group | ALL  | NULL          | NULL | NULL    | NULL |    3 | Using temporary; Using filesort |
	+----+-------------+-------------+------+---------------+------+---------+------+------+---------------------------------+
	1 row in set (0.00 sec)

去掉`group by`带来的`filesort`，`filesort`非常耗时。

	mysql> explain select * from table_group group by name order by null;
	+----+-------------+-------------+------+---------------+------+---------+------+------+-----------------+
	| id | select_type | table       | type | possible_keys | key  | key_len | ref  | rows | Extra           |
	+----+-------------+-------------+------+---------------+------+---------+------+------+-----------------+
	|  1 | SIMPLE      | table_group | ALL  | NULL          | NULL | NULL    | NULL |    3 | Using temporary |
	+----+-------------+-------------+------+---------------+------+---------+------+------+-----------------+
	1 row in set (0.00 sec)



## Point.5 优化 ORDER BY

order by子句使用索引先决条件

if（（order by 字段同为升序或者降序 ） &&  （where条件与order by使用相同的索引） && （order by的顺序与索引顺序相同））{

	may be use index
} else {

	not use index
}

order by 的字段混合 ASC 和 DESC

	SELECT * FROM t1 ORDER BY key_part1 DESC, key_part2 ASC；

用于查询行的关键字与 ORDER BY 中所使用的不相同

	SELECT * FROM t1 WHERE key2=constant ORDER BY key1；

对不同的关键字使用 ORDER BY（**这个没搞明白怎么回事**）

	SELECT * FROM t1 ORDER BY key1, key2；


## Point.6 优化OR条件

对于包含 OR 的查询子句，OR之间的两个条件列都必须用到索引，OR查询子句中的索引才会生效

	mysql> create table table_or(id int(11) primary key auto_increment,name char(59));
	Query OK, 0 rows affected (0.02 sec)

	mysql> insert into table_or(name) values('sujianhu9i'),('asdfa'),('zhaojianwei');
	Query OK, 3 rows affected (0.01 sec)
	Records: 3  Duplicates: 0  Warnings: 0


	mysql> select * from table_or;
	+----+-------------+
	| id | name        |
	+----+-------------+
	|  1 | sujianhu9i  |
	|  2 | asdfa       |
	|  3 | zhaojianwei |
	+----+-------------+
	3 rows in set (0.00 sec)

id有索引，name没有索引 select查询没有使用索引

	mysql> explain select * from table_or where id=4 or name='asd';
	+----+-------------+----------+------+---------------+------+---------+------+------+-------------+
	| id | select_type | table    | type | possible_keys | key  | key_len | ref  | rows | Extra       |
	+----+-------------+----------+------+---------------+------+---------+------+------+-------------+
	|  1 | SIMPLE      | table_or | ALL  | PRIMARY       | NULL | NULL    | NULL |    3 | Using where |
	+----+-------------+----------+------+---------------+------+---------+------+------+-------------+
	1 row in set (0.00 sec)


	mysql> create index on table_or(name);
	ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'on t
	ble_or(name())' at line 1
	mysql> create index sdfsdf on table_or(name);
	Query OK, 0 rows affected (0.02 sec)
	Records: 0  Duplicates: 0  Warnings: 0

题外话：貌似 MySQL version 8.0 不用显示指定索引名称，`create index on table_or(name)` 可以正确执行

	mysql> explain select * from table_or where id=4 or name='asd';
	+----+-------------+----------+-------+----------------+--------+---------+------+------+--------------------------+
	| id | select_type | table    | type  | possible_keys  | key    | key_len | ref  | rows | Extra                    |
	+----+-------------+----------+-------+----------------+--------+---------+------+------+--------------------------+
	|  1 | SIMPLE      | table_or | index | PRIMARY,sdfsdf | sdfsdf | 60      | NULL |    3 | Using where; Using index |
	+----+-------------+----------+-------+----------------+--------+---------+------+------+--------------------------+
	1 row in set (0.00 sec)

查询使用索引`sdfsdf`

## Point.7 优化嵌套查询

使用连接查询（JOIN）代替子查询。连接（JOIN）之所以更有效率一些，是因为 MySQL **不需要在内存中创建临时表**来完成这个逻辑上的需要两个步骤的查询工作。

	mysql > select name from user where id in （select uid from friends）

替代
	mysql > select name from user right join friends on user.id=friends.uid

备注：永远用小结果集驱动大结果集（Important）！


> 过滤后留下的结果集M,N(M>N)
> 1.如果都走全表的话，大表做驱动和小表做驱动都是M*N
> 2.如果走索引的话：
> a.索引对小表的作用不会太大，对于大表索引的作用就很大了，除非索引建的不好。。
> b.假设nexted-loop join中驱动表过滤后的行数为K，那么while(outer_row)一定会循环K次，这时驱动表上索引的功能是比聚簇索引占有更小的空间，一个节点上的数据量会更大些，减少随机I/O。
> c.如果被驱动表过滤后的行数为W，那么在while(outer_row)中两表连接条件上被驱动表还有机会利用索引来大大减少内循环的次数。
> 所以过滤结果中的小表做驱动表。。优化的目标是尽可能减少JOIN中Nested Loop的循环次数



## Point.8 使用SQL提示 HINT

MySQL在生成执行计划时，会根据自己的预估生成最优的执行计划，但是我们也可以人为干预执行计划的生成，性质有点类似与现实生活中的走后门，本来MySQL计划使用field_1字段的索引，人为提示MySQl使用field_2上的索引。下面是一些常用的SQL HINT

### use index

希望MySQL参考的索引

	mysql> show create table sel_type_no_index;

	CREATE TABLE `sel_type_no_index` (
	  `id` int(11) unsigned DEFAULT NULL,
	  `name` char(50) DEFAULT NULL,
	  KEY `id` (`id`),
	  KEY `name` (`name`)
	) ENGINE=InnoDB DEFAULT CHARSET=utf8

where条件中进行两次筛选，MySQL实际使用索引为`name`

	mysql> explain select * from sel_type_no_index where id>0 and name='ddd';
	+----+-------------+-------------------+------------+------+---------------+------+---------+-------+------+----------+-------------+
	| id | select_type | table             | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra       |
	+----+-------------+-------------------+------------+------+---------------+------+---------+-------+------+----------+-------------+
	|  1 | SIMPLE      | sel_type_no_index | NULL       | ref  | id,name       | name | 151     | const |    1 |   100.00 | Using where |
	+----+-------------+-------------------+------------+------+---------------+------+---------+-------+------+----------+-------------+
	1 row in set, 1 warning (0.00 sec)

提示使用索引`id`

	mysql> explain select * from sel_type_no_index use index(id) where id>0 and name='ddd';
	+----+-------------+-------------------+------------+-------+---------------+------+---------+------+------+----------+------------------------------------+
	| id | select_type | table             | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra                              |
	+----+-------------+-------------------+------------+-------+---------------+------+---------+------+------+----------+------------------------------------+
	|  1 | SIMPLE      | sel_type_no_index | NULL       | range | id            | id   | 5       | NULL |    1 |   100.00 | Using index condition; Using where |
	+----+-------------+-------------------+------------+-------+---------------+------+---------+------+------+----------+------------------------------------+
	1 row in set, 1 warning (0.00 sec)

### forec index

强制MySQL使用一个特定索引，引用上一个例子

	mysql> explain select * from sel_type_no_index force index(id) where id>0 and name='ddd';
	+----+-------------+-------------------+------------+-------+---------------+------+---------+------+------+----------+------------------------------------+
	| id | select_type | table             | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra                              |
	+----+-------------+-------------------+------------+-------+---------------+------+---------+------+------+----------+------------------------------------+
	|  1 | SIMPLE      | sel_type_no_index | NULL       | range | id            | id   | 5       | NULL |    1 |   100.00 | Using index condition; Using where |
	+----+-------------+-------------------+------------+-------+---------------+------+---------+------+------+----------+------------------------------------+
	1 row in set, 1 warning (0.00 sec)



### ignore index

	mysql> explain select * from user where id=200;
	+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
	| id | select_type | table | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
	+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
	|  1 | SIMPLE      | user  | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
	+----+-------------+-------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
	1 row in set, 1 warning (0.00 sec)

ignore index

	mysql> explain select * from user ignore index(primary) where id=200;
	+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
	| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
	+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
	|  1 | SIMPLE      | user  | NULL       | ALL  | NULL          | NULL | NULL    | NULL |  224 |     0.45 | Using where |
	+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
	1 row in set, 1 warning (0.00 sec)

### SQL_BUFFER_RESULT

强制生成mysql结果集，尽快释放锁定的表，适用于结果集传递耗时长的场景

	mysql> select SQL_BUFFER_RESULT * from user;

## Point.9 数据类型隐式转换

	SELECT * FROM `job_details` WHERE `date` = '20190122' ORDER BY `date` DESC LIMIT 1000;
	SELECT * FROM `job_details` WHERE `date` = 20190122 ORDER BY `date` DESC LIMIT 1000;

## 补充一个知识点:QUERY SQL 的执行顺序

1. **FORM**: 对FROM左边的表和右边的表计算笛卡尔积，产生虚表VT1。
1. **ON**: 对虚表VT1进行ON过滤，只有那些符合<join-condition>的行才会被记录在虚表VT2中。
1. **JOIN**： 如果指定了OUTER JOIN（比如left join、 right join），那么保留表中未匹配的行就会作为外部行添加到虚拟表VT2中，产生虚拟表VT3。
1. **WHERE**： 对虚拟表VT3进行WHERE条件过滤。只有符合<where-condition>的记录才会被插入到虚拟表VT4中。
1. **GROUP BY**: 根据group by子句中的列，对VT4中的记录进行分组操作，产生VT5。
1. **HAVING**： 对虚拟表VT5应用having过滤，只有符合<having-condition>的记录才会被 插入到虚拟表VT6中。
1. **SELECT**： 执行select操作，选择指定的列，插入到虚拟表VT7中。
1. **DISTINCT**： 对VT7中的记录进行去重。产生虚拟表VT8.
1. **ORDER BY**: 将虚拟表VT8中的记录按照<order_by_list>进行排序操作，产生虚拟表VT9.

### 虚拟表、临时表、内存表、视图

#### 虚拟表
逻辑上存在，实际上并不存在，抽象概念帮助理解执行计划

#### 临时表
当工作在十分大的表上运行时，在实际操作中你可能会需要运行很多的相关查询，来获的一个大量数据的小的子集。较好的办法，不是对整个表运行这些查询，而是让MySQL每次找出所需的少数记录，将记录选择到一个临时表，然后对这些表运行查询。

创建临时表

(1)定义字段

	CREATE TEMPORARY TABLE tmp_table (
	name VARCHAR(10) NOT NULL,
	value INTEGER NOT NULL)

(2)直接将查询结果导入临时表

	CREATE TEMPORARY TABLE tmp_table SELECT * FROM table_name

(3)查询临时表

	select * from tmp_table
(4)删除临时表

	drop table tmp_table

[参考博文](https://yypiao.iteye.com/blog/2359859 "内存表虚拟表临时表")
#### 视图
预编译的SQL语句，并不保存实际数据

#### 内存表
表结构建在磁盘里，数据在内存里 ，当停止服务后，表中的数据丢失，而表的结构不会丢失。

## 小结

在索引使用方面，语句越简单越好，这样**执行计划稳定**，一定要使用绑定变量，减少语句解析，尽量减少表关联，尽量减少分布式事务。用户并发大，用户的请求十分密集时，批量更新要分批进行，避免堵塞。
