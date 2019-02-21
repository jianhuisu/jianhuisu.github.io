---
title : 常见实用SQL优化技巧
tags :
	- MySQL
---

## Point.1 定期检查表，分析表

批量导入数据的优化

插入数据的优化

优化 GROUP BY

优化 ORDER BY

优化OR条件

优化嵌套查询

## Point.1 使用SQL提示 HINT
	
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
	


