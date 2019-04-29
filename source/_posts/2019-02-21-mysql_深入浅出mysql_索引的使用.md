---
title : 如何使用索引才能生效
categories : 
 - mysql 
tags :
	- MySQL
---

## Question.1 索引是什么？

索引是一种数据结构	 帮助快速进行数据的检索，对相关列使用索引是提高SELECT性能的最佳途径。

## Question.2 常见索引有哪些？

常见存储引擎支持的索引存储类型

- MyISAM   ：BTREE
- Innodb   ：BTREE
- MEMORY   ：BTREE、HASH
- HEAP     ：BTREE、HASH

常见索引类型

- 普通索引
- 复合索引
- 唯一索引
- 主键索引
- 前缀索引
- 全文索引
- 合成索引

## Question.3 如何选择为哪些列加索引

- where条件中用到的字段可以考虑创建索引
- 字段取值范围基数较大的

## Question.4 如何选择索引的类型

常见索引这里不赘述，这里有个合成索引值得推荐一下。
采用合成索引的方式可以提高大文本字段的匹配性能，这种思想是在对大文本字段存储时，采用一个单独的字段存储该内容的散列值，需要检索文本的时候，间接的检索散列值即可。

	Create Table: CREATE TABLE `text_md5` (
	  `id` varchar(100) DEFAULT NULL,
	  `content` text,
	  `hash` char(100) DEFAULT NULL
	) ENGINE=MyISAM DEFAULT CHARSET=utf8

	mysql> insert into text_md5(id,content,hash) values(1,repeat('sujianhui 2222',2),md5(content));
	Query OK, 1 row affected (0.02 sec)

	mysql> insert into text_md5(id,content,hash) values(1,repeat('sujianhui 1111',2),md5(content));
	Query OK, 1 row affected (0.04 sec)

	mysql> insert into text_md5(id,content,hash) values(2,repeat('sujianhui',2),md5(content));
	Query OK, 1 row affected (0.04 sec)

看一下存储情况

	mysql> select * from text_md5;
	+------+------------------------------+----------------------------------+
	| id   | content                      | hash                             |
	+------+------------------------------+----------------------------------+
	| 1    | sujianhui 2222sujianhui 2222 | fefbbf5186b7da53015f5076a8daca12 |
	| 1    | sujianhui 1111sujianhui 1111 | 6e597cf3ae70674e3f744f9c9b3b4a08 |
	| 2    | sujianhuisujianhui           | 3bba68dd0b6895f4d1d11fec8fa8e8aa |
	+------+------------------------------+----------------------------------+

使用散列值进行精准匹配

	mysql> select id,content from text_md5 where hash=md5(repeat('sujianhui',2));
	+------+--------------------+
	| id   | content            |
	+------+--------------------+
	| 2    | sujianhuisujianhui |
	+------+--------------------+

这种**合成索引**的思想真的让我眼前一亮，大为叹服。
mysql的BLOB/CLOB字段还支持**前缀索引**，就是为字段的前N个字节创建索引（注意%不能放在前面）

## Question.5 添加了索引就一定会生效吗

不一定，以下几种情况执行计划不会使用索引

1. 使用like进行模糊查询时，`%` 位于首位，索引不生效。
1. 使用or连接条件时，如果两个条件中有一个关键字没有使用索引，则索引不生效。
1. 使用MEMORY/HEAP存储引擎的表中不使用`=`进行数据索引，则索引不生效，HASH类型索引无法进行除`=`的其他比较。
1. 复合索引中的第一列没有被使用，则复合索引不生效。
1. 如果列类型是字符串，但是检索时常数使用整型，索引不生效，例如 create_time 字段类型为 char(10) , `where create_time = 201902` 不使用索引，`where create_time = '201902'` 使用索引。

## Question.6 如何查看索引的使用情况

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

