---
title : SQL分析工具EXPLAIN详解
tags : 
	- MySQL
---

## 概览

使用 explain + QUERY_SQL
	

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
							可能值： 
							 	SIMPLE  简单表
							    PRIMARY  
								UNION
								SUBQUERY

	table:             输出结果集的表
	partitions:        
	type:              表的连接类型
							可能值：
								// 从上到下 效率由高到低
								system 
								const
								eq_ref  
								ref  
								fulltext  
								ref_or_null  
								index_merge 
								unique_subquery  
								index_subquery  
								range  
								index 
								ALL
	 
	possible_keys:     可能会使用到的索引
	key:               实际使用到的索引
	key_len:           索引字段的长度
	ref:               
	rows:              扫描的行数
	filtered:          
	extra:             执行情况的说明与描述


以上参数中，分析执行效率高低的关键指标

- type : 至少达到 range级别 ，否则会影响性能
- key :  实际使用索引
- key_len :索引长度
- rows : 扫描行数


## 详解

### Params.1 select_type

指明各单位select查询的查询类型, 定位语句位置

### Params.2 type

创建表 sel_type_child
 
	CREATE TABLE IF NOT EXISTS `sel_type_child` (
	  `id` int(10) NOT NULL AUTO_INCREMENT,
	  `name` char(50) DEFAULT NULL,
	  `pid` int(11) DEFAULT NULL,
	  PRIMARY KEY (`id`)
	) ENGINE=MyISAM AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;
	

	INSERT INTO `sel_type_child` (`id`, `name`, `pid`) VALUES(1, '张学良', 1),(2, '李世民', 2),(3, '雍正', 3);
	
	
创建表 sel_type_parent

	CREATE TABLE IF NOT EXISTS `sel_type_parent` (
	  `id` int(10) NOT NULL AUTO_INCREMENT,
	  `name` char(50) DEFAULT NULL,
	  PRIMARY KEY (`id`)
	) ENGINE=MyISAM AUTO_INCREMENT=5 DEFAULT CHARSET=utf8;
	
	INSERT INTO `sel_type_parent` (`id`, `name`) VALUES(1, '张作霖'),(2, '李渊'),(3, '康熙'),(4, '曾国藩');

#### system

定义：当查询表为常量表时，查询类型为 `system`
常量表 const table 的定义: const table 指的是满足下面四个条件的表

1. 当查询条件中包含了某个表的主键或者非空的索引列
1. 该列的判定条件为等值条件
1. 目标值的类型与该列的类型一致
1. 目标值为一个确定的常量


	mysql> create table sel_type_index(id int(10),name char(50))engine=MyISAM charset=utf8;
	Query OK, 0 rows affected, 1 warning (0.13 sec)
	
	mysql> create index on sel_type_index(id);
	Query OK, 0 rows affected (0.15 sec)
	Records: 0  Duplicates: 0  Warnings: 0
	
	mysql> insert into sel_type_index(id,name) values(1,'name_1');
	Query OK, 1 row affected (0.01 sec)
	
	mysql> explain select * from sel_type_index where id=1;
	+----+-------------+----------------+------------+--------+---------------+------+---------+------+------+----------+-------+
	| id | select_type | table          | partitions | type   | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
	+----+-------------+----------------+------------+--------+---------------+------+---------+------+------+----------+-------+
	|  1 | SIMPLE      | sel_type_index | NULL       | system | id            | NULL | NULL    | NULL |    1 |   100.00 | NULL  |
	+----+-------------+----------------+------------+--------+---------------+------+---------+------+------+----------+-------+
	1 row in set, 1 warning (0.00 sec)

这个没什么好说的，一条数据没有太大的优化价值

#### const

	mysql> explain select * from sel_type_parent where id=1;
	+----+-------------+-----------------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
	| id | select_type | table           | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
	+----+-------------+-----------------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
	|  1 | SIMPLE      | sel_type_parent | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
	+----+-------------+-----------------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
	1 row in set, 1 warning (0.00 sec)


#### ref

使用普通索引
`=` 运算符 

	mysql> create table sel_type_index(id int(10),name char(50))engine=MyISAM charset=utf8;
	Query OK, 0 rows affected, 1 warning (0.13 sec)
	
	mysql> create index on sel_type_index(id);
	Query OK, 0 rows affected (0.15 sec)
	Records: 0  Duplicates: 0  Warnings: 0
	
	mysql> insert into sel_type_index(id,name) values(2,'name_2'),(3,'name_3'),(4,'name_4');
	Query OK, 3 rows affected (0.00 sec)
	Records: 3  Duplicates: 0  Warnings: 0
	
	mysql> explain select * from sel_type_index where id=1;
	+----+-------------+----------------+------------+------+---------------+------+---------+-------+------+----------+-------+
	| id | select_type | table          | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra |
	+----+-------------+----------------+------------+------+---------------+------+---------+-------+------+----------+-------+
	|  1 | SIMPLE      | sel_type_index | NULL       | ref  | id            | id   | 5       | const |    1 |   100.00 | NULL  |
	+----+-------------+----------------+------------+------+---------------+------+---------+-------+------+----------+-------+
	1 row in set, 1 warning (0.00 sec)

#### ref_or_null

条件中包含对null的查询

#### range

使用了 `>`,`<`,`IN`,`NOT IN`,`between ... and ...` 范围比较

	mysql> explain select * from sel_type_parent where id>1;
	+----+-------------+-----------------+------------+-------+---------------+---------+---------+------+------+----------+-----------------------+
	| id | select_type | table           | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra                 |
	+----+-------------+-----------------+------------+-------+---------------+---------+---------+------+------+----------+-----------------------+
	|  1 | SIMPLE      | sel_type_parent | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |    3 |   100.00 | Using index condition |
	+----+-------------+-----------------+------------+-------+---------------+---------+---------+------+------+----------+-----------------------+
	1 row in set, 1 warning (0.00 sec)

### index

对于每一行，都是通过查询索引得出


#### all

没有使用索引，全表扫描

	mysql> create table sel_type_no_index(id int(11),name char(50));
	Query OK, 0 rows affected (0.14 sec)
	
	mysql> insert into sel_type_no_index(id,name) values(1,'name_1');
	Query OK, 1 row affected (0.07 sec)
	
	mysql> select * from sel_type_no_index where id=1;
	+------+--------+
	| id   | name   |
	+------+--------+
	|    1 | name_1 |
	+------+--------+
	1 row in set (0.00 sec)
	
	mysql> explain select * from sel_type_no_index where id=1;
	+----+-------------+-------------------+------------+------+---------------+------+---------+------+------+----------+-------------+
	| id | select_type | table             | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
	+----+-------------+-------------------+------------+------+---------------+------+---------+------+------+----------+-------------+
	|  1 | SIMPLE      | sel_type_no_index | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    1 |   100.00 | Using where |
	+----+-------------+-------------------+------------+------+---------------+------+---------+------+------+----------+-------------+
	1 row in set, 1 warning (0.00 sec)


备注： 总结来讲

	 等于操作 > 比较操作
	 唯一索引 > 普通索引 > 没有使用索引(这并不意味着索引越多越好)
	 
### Params.3 key 


### Params.4 key_len

越短越好

### Params.5 rows

扫描的行数，越少越好
 
## 总结
	
索引的本质为数据结构




