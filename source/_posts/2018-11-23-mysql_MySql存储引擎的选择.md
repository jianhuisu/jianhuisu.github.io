---
title : MySQL存储引擎的选择
tags : 
	- 深入浅出MySQL
keywords : "深入浅出MySQL,mysql"
---

每次建表的时候心里都发虚，不知道去如何选择数据引擎、字段类型.才能贴合业务，获得良好的性能。每次都要去翻看前辈是如何设置的，这种不自信的感觉让我很不爽。

SQL 结构化查询语言

#### MyISAM
	表锁
	不支持事务
	不支持外键

适用场景：以select、insert为主的服务，对事务完整性没有要求的系统
备注：MyISAM存储引擎表支持三种存储格式

静态表

是MyISAM的默认存储格式。以固定空间存储数据，空余部分使用空格填充。实际存储内容尾部空格会被忽略。
查询、存储效率高，但是浪费空间，出现故障相对容易恢复

动态表

存储空间根据实际数据大小划分，相当于C中的动态分配内存，容易产生内存碎片。所以需要定期使用 `OPTMIZE TABLE` 或 `myisamchk -r` 命令改善性能，出现故障时恢复难度较大 

压缩存储



#### Innodb
	行锁
	支持事务
	支持外键
适用场景：update、delete为主，对事务完整性有要求的系统。例如计费系统

#### Memory
	表锁
	不支持事务
	不支持外键
适用场景：存储在RAM中，服务重启后数据清空。表存储大小有限制 一般为 16M。适合较小中间表的临时存储

## 如何正确的选取数据类型

正确的选择存储引擎之后，还需要正确的选取数据类型

当表引擎是 MyISAM时

char(4) 与 varchar(4) 的区别是  char长度固定，所以很快

#### char 与 varchar
- **MyISAM存储引擎**
当字段长度变化小时，建议采用char代替varchar，固定数据长度将带来一定的检索优势
- **InnoDB存储引擎**
建议varchar代替char，InnoDB在数据存储过程中，没有区分固定长度与变长序列，通俗点将就是，对char类型也会采取varchar类型的分配方式，而且char所占空间要大，
- **Memory存储引擎**
char 、varchar没有区别，因为目前该引擎使用固定的数据长度存储，最终都会以char类型处理
#### text与blog

##### 性能问题
将 text、blog字段删除时，虽然表中对应的数据已经消失，但是字段所占用的物理空间并没有释放

	mysql> create table text_table(id varchar(100),content text)engine=MyISAM charset=utf8;
	mysql> insert into text_table(id,content) value(1,repeat('hah',1000)); // 这条语句执行 10次
	mysql> insert into text_table(id,content) value(9,repeat('hah',10000)); // 这条语句执行 10次

查看一下data文件占用的空间大小

	[root@vagrant-centos65 qq]# du -sh text_table*
	4.0K	text_table_620.sdi
	172K	text_table.MYD
	4.0K	text_table.MYI

删除表中部分数据，然后再次查看占用物理空间，没有变化：
	
	mysql> delete from text_table where id=9;

	[root@vagrant-centos65 qq]# du -sh text_table*
	4.0K	text_table_620.sdi
	172K	text_table.MYD
	4.0K	text_table.MYI

即使重启了mysql服务，依然没有变化：
	
	[root@vagrant-centos65 qq]# service mysqld restart
	Stopping mysqld:                                           [  OK  ]
	Starting mysqld:                                           [  OK  ]

	[root@vagrant-centos65 qq]# du -sh text_table*
	4.0K	text_table_620.sdi
	172K	text_table.MYD
	4.0K	text_table.MYI

对mysql表进行优化 `optimize table text_table`，这时，删除造成的空间空洞才真正被释放：
	
	mysql> optimize table text_table;
	
	+---------------+----------+----------+----------+
	| Table         | Op       | Msg_type | Msg_text |
	+---------------+----------+----------+----------+
	| qq.text_table | optimize | status   | OK       |
	+---------------+----------+----------+----------+
	1 row in set (0.05 sec)
	
	...

	[root@vagrant-centos65 qq]# du -sh text_table*
	4.0K	text_table_620.sdi
	56K	text_table.MYD
	4.0K	text_table.MYI

###### 匹配问题

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

	mysql> alter table text_md5 change content content blob;
	mysql> create index idx_blob on text_md5(content(100));

	mysql> explain select * from text_md5 where content like "sujianhui%";
	+----+-------------+----------+------------+-------+---------------+----------+---------+------+------+----------+-------------+
	| id | select_type | table    | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra       |
	+----+-------------+----------+------------+-------+---------------+----------+---------+------+------+----------+-------------+
	|  1 | SIMPLE      | text_md5 | NULL       | range | idx_blob      | idx_blob | 103     | NULL |    2 |   100.00 | Using where |
	+----+-------------+----------+------------+-------+---------------+----------+---------+------+------+----------+-------------+

type为`range`级别，勉强及格。

#### 浮点数与定点数

float   浮点数 超过限定格式时，会进行四舍五入处理 float(m,d) 最多保留`m - n`位整数，小数部分有效数字为n位
decimal 定点数 实际上以字符串形式存储数值，不会有精度损失  （货币应用中）

特别需要注意的一点，编程中要尽量避免浮点数的比较，如果不可避免，也尽量使用范围比较代替 ==

*题外话：我从开源产品edusoho订单模块浮点数的使用，将浮点数转化为整数后再进行操作，但是我没有搞明白这样做会比两个浮点数直接比较准确，浮点数与整数相乘结果仍然为浮点数，感觉没有什么意义或者自己没有领会到位。等我把`ZVAL`转换摸透再来完善这里*

	<?php	
	
		$f1 = 1.23;
		$f2 = 1.20;
		
		if($f1 * 100  == $f2*100 ){
		    echo 1;
		}else{
		    echo 2;
		}

浮点数使用注意问题

- 尽量避免浮点数比较
- 货币等对精度敏感的数据要使用定点数存储
- 使用浮点数注意精度损失
	

##### 时间数据类型

这块没什么意思，不说了

## 参考资料

深入浅出MySQL第三版