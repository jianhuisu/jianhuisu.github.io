---
title : MySQL存储引擎的选择
categories : 
 - mysql 
tags :
	- MySQL
keywords : "深入浅出MySQL,mysql"
---

每次建表的时候心里都发虚，不知道去如何选择数据引擎、字段类型.才能贴合业务，获得良好的性能。
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

#### char 与 varchar
- **MyISAM存储引擎**
当字段长度变化小时，建议采用char代替varchar，固定数据长度将带来一定的检索优势
- **InnoDB存储引擎**
建议varchar代替char，InnoDB在数据存储过程中，没有区分固定长度与变长序列，通俗点将就是，对char类型也会采取varchar类型的分配方式，而且char所占空间要大，
- **Memory存储引擎**
char 、varchar没有区别，因为目前该引擎使用固定的数据长度存储，最终都会以char类型处理

Tips:对于上面InnoDB存储引擎优先选择varchar代替char的实验，将同样量级、同样内容的数据存入同样结构的表中，查看表文件所占磁盘空间

	[root@vagrant-centos65 study]# ls -lhS
	total 117M
	-rw-r----- 1 root root  64M Mar 27 13:44 myisam_char_255_table.MYD
	-rw-r----- 1 root root  26M Mar 27 16:13 innodb_char_255_table.ibd
	-rw-r----- 1 root root  11M Mar 27 16:12 innodb_char_10_table.ibd
	-rw-r----- 1 root root  11M Mar 27 16:14 innodb_varchar_255_table.ibd
	-rw-r----- 1 root root 2.6M Mar 27 13:43 myisam_char_10_table.MYD
	-rw-r----- 1 root root 2.2M Mar 27 16:02 myisam_varchar_255_table.MYD
	-rw-r----- 1 root root 1.0K Mar 27 16:08 myisam_char_10_table.MYI
	-rw-r----- 1 root root 1.0K Mar 27 16:08 myisam_char_255_table.MYI
	-rw-r----- 1 root root 1.0K Mar 27 16:08 myisam_varchar_255_table.MYI

因为事务的缘故，innodb存储引擎表在对varchar类型、char类型字段进行数据存储时所执行的存储过程并无不同，即效率相同。但是char类型字段要比varchar类型占用更多的空间。

	innodb_char_255_table.ibd > innodb_varchar_255_table.ibd

在同样量级、内容、结构的条件下，innodb表要比myisam占用更多的空间

	innodb_char_10_table.ibd > myisam_char_10_table.MYD

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

### 外键约束

外键的作用：从数据库层面保障数据的一致性与完整性

使用场景：系统的权限管理、金融系统

eg.

	// 事业部表
	mysql> CREATE TABLE `business` (
	  `business_id` int(10) NOT NULL AUTO_INCREMENT,
	  `name` varchar(50) DEFAULT NULL,
	  `update_time` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
	  PRIMARY KEY (`business_id`)
	) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8

	// 军团表
	CREATE TABLE `region` (
	  `region_id` int(11) NOT NULL,
	  `name` varchar(50) DEFAULT NULL,
	  `business_id` int(11) DEFAULT NULL,
	  `update_time` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
	  PRIMARY KEY (`region_id`),
	  KEY `fk_region_business` (`business_id`),
	  CONSTRAINT `fk_region_business` FOREIGN KEY (`business_id`) REFERENCES `business` (`business_id`)
	) ENGINE=InnoDB DEFAULT CHARSET=utf8

声明方式: CONSTRAINT `fk_region_business` FOREIGN KEY (`business_id`) REFERENCES `business` (`business_id`)

- 当删除`business` 表中数据时，若子表`region`存在关联数据，禁止删除操作
- 当增加`region` 表中数据时，若父表`business`不存在关联数据，禁止增加操作

#### 外键关联模式

- 当某个表的主键被其它表创建了外键参照，那么该表的对应索引与主键禁止删除
- RESTRICT: 限制子表中有关联记录父表禁止删除
- CASCADE : 父表更新记录时，更新子表中相关记录
- SET NULL: 父表更新记录时，子表中相关记录置为NULL

eg. CONSTRAINT `fk_region_business` FOREIGN KEY (`business_id`) REFERENCES `business` (`business_id`) ON DELETE RESTRICT ON UPDATE CASCADE;

#### 如何导入含有外键的数据

SET FOREIGN_KEY_CHECKS=0; # 临时关闭线程中外键检查
数据导入命令
SET FOREIGN_KEY_CHECKS=1;#  恢复线程中外键检查

eg.

	SET FOREIGN_KEY_CHECKS=0;

	CREATE TABLE IF NOT EXISTS `region` (
	  `region_id` int(11) NOT NULL,
	  `name` varchar(50) DEFAULT NULL,
	  `business_id` int(11) DEFAULT NULL,
	  `update_time` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
	  PRIMARY KEY (`region_id`),
	  KEY `fk_region_business` (`business_id`),
	  CONSTRAINT `fk_region_business` FOREIGN KEY (`business_id`) REFERENCES `business` (`business_id`)
	) ENGINE=InnoDB DEFAULT CHARSET=utf8;

	DELETE FROM `region`;
	/*!40000 ALTER TABLE `region` DISABLE KEYS */;
	INSERT INTO `region` (`region_id`, `name`, `business_id`, `update_time`) VALUES
		(1, '第一军团', 1, '2019-01-10 20:19:04'),
		(2, '第二十一军团', 2, '2019-01-10 20:19:49'),
		(3, '第三十一军团', 3, '2019-01-10 20:20:10');
	/*!40000 ALTER TABLE `region` ENABLE KEYS */;
	/*!40101 SET SQL_MODE=IFNULL(@OLD_SQL_MODE, '') */;
	/*!40014 SET FOREIGN_KEY_CHECKS=IF(@OLD_FOREIGN_KEY_CHECKS IS NULL, 1, @OLD_FOREIGN_KEY_CHECKS) */;
	/*!40101 SET CHARACTER_SET_CLIENT=@OLD_CHARACTER_SET_CLIENT */;

	CREATE TABLE IF NOT EXISTS `business` (
	  `business_id` int(10) NOT NULL AUTO_INCREMENT,
	  `name` varchar(50) DEFAULT NULL,
	  `update_time` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
	  PRIMARY KEY (`business_id`)
	) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;

	SET FOREIGN_KEY_CHECKS=1;

注意：使用上述命令可以忽略数据的导入顺序，但是如果缺少父表，依然无法成功导入。


## MERGE

alter table articles modify id int ; 【重新定义列类型】

alter table articles drop primary key;


## 参考资料

深入浅出MySQL第三版
