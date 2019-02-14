---
title : MySQL中的常用函数
tags :
	- MySQL
	- 常用函数 
---

### 以下示例均操作user表

	mysql> show create table user\G
	*************************** 1. row ***************************
	       Table: user
	Create Table: CREATE TABLE `user` (
	  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
	  `name` char(50) DEFAULT NULL,
	  PRIMARY KEY (`id`)
	) ENGINE=MyISAM AUTO_INCREMENT=9 DEFAULT CHARSET=utf8
	1 row in set (0.00 sec)

表中数据
	
	mysql> select * from user;
	+----+---------------+
	| id | name          |
	+----+---------------+
	|  1 | sujianhui     |
	|  2 | zhaojianwei   |
	|  3 | zhnangwenyuan |
	|  4 | fuqiang       |
	|  5 | sujianhui_1   |
	|  6 | sujianhui_2   |
	|  7 |               |
	|  8 | NULL          |
	+----+---------------+
	8 rows in set (0.00 sec)

### 字符串函数
CONCAT(name1,name2,...)

	mysql> select concat(1,name),name from user;
	+----------------+---------------+
	| concat(1,name) | name          |
	+----------------+---------------+
	| 1sujianhui     | sujianhui     |
	| 1zhaojianwei   | zhaojianwei   |
	| 1zhnangwenyuan | zhnangwenyuan |
	| 1fuqiang       | fuqiang       |
	| 1sujianhui_1   | sujianhui_1   |
	| 1sujianhui_2   | sujianhui_2   |
	| 1              |               |
	| NULL           | NULL          |
	+----------------+---------------+
	8 rows in set (0.00 sec)
	
	mysql> select concat(1,id,name),name from user;
	+-------------------+---------------+
	| concat(1,id,name) | name          |
	+-------------------+---------------+
	| 11sujianhui       | sujianhui     |
	| 12zhaojianwei     | zhaojianwei   |
	| 13zhnangwenyuan   | zhnangwenyuan |
	| 14fuqiang         | fuqiang       |
	| 15sujianhui_1     | sujianhui_1   |
	| 16sujianhui_2     | sujianhui_2   |
	| 17                |               |
	| NULL              | NULL          |
	+-------------------+---------------+
	8 rows in set (0.00 sec)

注意：操作项中含有NULL值，那么返回值也为NULL

LEFT(str,n)
RIGHT(str,n)

	mysql> select left(name,2),right(name,2) from user where id=1;
	+--------------+---------------+
	| left(name,2) | right(name,2) |
	+--------------+---------------+
	| su           | ui            |
	+--------------+---------------+
	1 row in set (0.00 sec)

LTRIM(str)
RTRIM(str)
TRIM(str)

SUBSTRING(str,x,y):从第x位开始y个字符

	mysql> select substring(name,1,3) from user where id=1;
	+---------------------+
	| substring(name,1,3) |
	+---------------------+
	| suj                 |
	+---------------------+
	1 row in set (0.00 sec)
	
REPLACE(str，a，b)  将字符串str中所有的字符串a替换为字符串b

	mysql> select replace(name,'zhao','su') from user;
	+---------------------------+
	| replace(name,'zhao','su') |
	+---------------------------+
	| sujianhui                 |
	| sujianwei                 |
	| zhnangwenyuan             |
	| fuqiang                   |
	| sujianhui_1               |
	| sujianhui_2               |
	|                           |
	| NULL                      |
	+---------------------------+
	8 rows in set (0.00 sec)

INSERT(str,x,y,a)：将指定字符串str从第x字符开始，y长度的字符子串替换为字符串a

	mysql> select insert(name,1,4,'su') from user;
	+-----------------------+
	| insert(name,1,4,'su') |
	+-----------------------+
	| suanhui               |
	| sujianwei             |
	| sungwenyuan           |
	| suang                 |
	| suanhui_1             |
	| suanhui_2             |
	|                       |
	| NULL                  |
	+-----------------------+
	8 rows in set (0.00 sec)

LPAD(str,n,pad)
RPAD(str,n,pad)：将字符串str以pad填充到指定长度n

	mysql> select lpad(name,5,'a') from user where id=7;
	+------------------+
	| lpad(name,5,'a') |
	+------------------+
	| aaaaa            |
	+------------------+
	1 row in set (0.00 sec)

REPEAT(str,n)

	mysql> select repeat('su',11);
	+------------------------+
	| repeat('su',11)        |
	+------------------------+
	| sususususususususususu |
	+------------------------+
	1 row in set (0.00 sec)

LOWWER(str)
UPPER(str)
STRCMP(str1,str2):比较字符串str1与str2的ANSII码值大小。相等返回0，s1>s2 返回1，s1<s2 返回-1
LOCATE 返回字符在字段中第一次出现的位置 没有出现则返回0 select LOCATE('第一军团',name) from region

### 数值函数

- RAND()     返回小于1的随机数
- MOD(a,b)   求模
- FLOOR(x)    舍去取整
- CEIL(x)     进一取整
- ROUND(m,n) 四舍五入
- TRUNCATE(m,n) 截断保存小数

### 时间函数

CURDATE() 返回当前年月日
CURTIME() 返回当前时间
NOW()     返回当前完整时间  

	mysql> select curdate(),curtime(),now();
	+------------+-----------+---------------------+
	| curdate()  | curtime() | now()               |
	+------------+-----------+---------------------+
	| 2018-12-29 | 12:18:24  | 2018-12-29 12:18:24 |
	+------------+-----------+---------------------+
	1 row in set (0.05 sec)

UNIX_TIMESTAMP('2018-12-12'):将日期格式字符串转化为时间戳
FROM_UNIXTIME(1543981584)   ：将时间戳转化为日期格式字符串

	mysql> select UNIX_TIMESTAMP('2018-12-12'),FROM_UNIXTIME(1543981584);
	+------------------------------+---------------------------+
	| UNIX_TIMESTAMP('2018-12-12') | FROM_UNIXTIME(1543981584) |
	+------------------------------+---------------------------+
	|                   1544544000 | 2018-12-05 11:46:24       |
	+------------------------------+---------------------------+
	1 row in set (0.00 sec)

MONTHNAME(date)	 返回月份的英文名 
WEEK(date)
YEAR(date)
HOUR(date)

	mysql> select week('2018-12-12'),year('2018-12-12'),hour('2018-12-12 12:00:00');
	+--------------------+--------------------+-----------------------------+
	| week('2018-12-12') | year('2018-12-12') | hour('2018-12-12 12:00:00') |
	+--------------------+--------------------+-----------------------------+
	|                 49 |               2018 |                          12 |
	+--------------------+--------------------+-----------------------------+
	1 row in set (0.00 sec)

DATE_FORMATE(date,fmt)	

DATE_ADD():计算当前时间指定时间段后日期。。
DATE_DIFF():计算两个时间点之间相差天数

	mysql> select date_add(now(),INTERVAL 31 day) afterdays,now();
	+---------------------+---------------------+
	| afterdays           | now()               |
	+---------------------+---------------------+
	| 2019-01-29 13:33:18 | 2018-12-29 13:33:18 |
	+---------------------+---------------------+
	1 row in set (0.00 sec)
	
	mysql> 
	mysql> 
	mysql> 
	mysql> select date_diff('2018-12-29','2019-02-04');
	ERROR 1305 (42000): FUNCTION qq.date_diff does not exist
	mysql> select datediff('2018-12-29','2019-02-04');
	+-------------------------------------+
	| datediff('2018-12-29','2019-02-04') |
	+-------------------------------------+
	|                                 -37 |
	+-------------------------------------+
	1 row in set (0.00 sec)


### 流程函数

IF(ifvalue,value1,value2)

	mysql> select if(name='','my name is empty',name) from user;
	+-------------------------------------+
	| if(name='','my name is empty',name) |
	+-------------------------------------+
	| sujianhui                           |
	| zhaojianwei                         |
	| zhnangwenyuan                       |
	| fuqiang                             |
	| sujianhui_1                         |
	| sujianhui_2                         |
	| my name is empty                    |
	| NULL                                |
	+-------------------------------------+
	8 rows in set (0.00 sec)


IFNULL(ifvalues,value1)

	mysql> select ifnull(name,'my name is null') from user;
	
	+--------------------------------+
	| ifnull(name,'my name is null') |
	+--------------------------------+
	| sujianhui                      |
	| zhaojianwei                    |
	| zhnangwenyuan                  |
	| fuqiang                        |
	| sujianhui_1                    |
	| sujianhui_2                    |
	|                                |
	| my name is null                |
	+--------------------------------+
	8 rows in set (0.00 sec)

case when exp then ... else ... end

	mysql> select case when name is null then 'name is null' when name='' then 'name is empty' else name end from user;
	+--------------------------------------------------------------------------------------------+
	| case when name is null then 'name is null' when name='' then 'name is empty' else name end |
	+--------------------------------------------------------------------------------------------+
	| sujianhui                                                                                  |
	| zhaojianwei                                                                                |
	| zhnangwenyuan                                                                              |
	| fuqiang                                                                                    |
	| sujianhui_1                                                                                |
	| sujianhui_2                                                                                |
	| name is empty                                                                              |
	| name is null                                                                               |
	+--------------------------------------------------------------------------------------------+
	8 rows in set (0.01 sec)

case ... when exp then ... else ... end，这个函数在进行数据拆分重组时非常有用。举个例子：我上学那会，六年级属于初中的一年级，五年级为小学的最后一级,创建数据表存储层级关系:
	
	mysql> create table origanization(
		id int(11) auto_increment primary key,
		org1 char(50),
		org2 char(50),
		org3 char(50)
	);
	Query OK, 0 rows affected (0.26 sec)
		
	mysql> desc origanization;
	+-------+----------+------+-----+---------+----------------+
	| Field | Type     | Null | Key | Default | Extra          |
	+-------+----------+------+-----+---------+----------------+
	| id    | int(11)  | NO   | PRI | NULL    | auto_increment |
	| org1  | char(50) | YES  |     | NULL    |                |
	| org2  | char(50) | YES  |     | NULL    |                |
	| org3  | char(50) | YES  |     | NULL    |                |
	+-------+----------+------+-----+---------+----------------+
	4 rows in set (0.00 sec)

将阶段、年级、班级数据存入表	
	mysql> insert into origanization(org1,org2,org3) 
		values('小学','五年级','一班'),
		('小学','五年级','2班'),
		('初中','六年级','一班'),
		('初中','六年级','2班'),
		('高中','高一','一班'),
		('高中','高一','2班');
	Query OK, 6 rows affected (0.18 sec)
	Records: 6  Duplicates: 0  Warnings: 0
	
	mysql> select * from origanization;
	+----+--------+-----------+--------+
	| id | org1   | org2      | org3   |
	+----+--------+-----------+--------+
	|  1 | 小学   | 五年级    | 一班   |
	|  2 | 小学   | 五年级    | 2班    |
	|  3 | 初中   | 六年级    | 一班   |
	|  4 | 初中   | 六年级    | 2班    |
	|  5 | 高中   | 高一      | 一班   |
	|  6 | 高中   | 高一      | 2班    |
	+----+--------+-----------+--------+
	6 rows in set (0.00 sec)

但是等我弟弟上学时，六年级被划分成为小学的最后一级,因为种种原因,不能更改原数据，但是需要查看新的划分关系，可以使用case函数生成中间表：
	
	mysql> select case org2 when '六年级' then '小学' else org1 end as new_org1,org2,org3 from origanization;
	+----------+-----------+--------+
	| new_org1 | org2      | org3   |
	+----------+-----------+--------+
	| 小学     | 五年级    | 一班   |
	| 小学     | 五年级    | 2班    |
	| 小学     | 六年级    | 一班   |
	| 小学     | 六年级    | 2班    |
	| 高中     | 高一      | 一班   |
	| 高中     | 高一      | 2班    |
	+----------+-----------+--------+
	6 rows in set (0.00 sec)

特别注意，case后的字段不能直接在where条件中使用，如需使用需要以中间表的形式引用
	
	mysql> select case org2 when '六年级' then '小学' else org1 end as new_org1,org2,org3 from origanization where new_org1='小学';
	ERROR 1054 (42S22): Unknown column 'new_org1' in 'where clause'
	 

### 其它函数

- DATABASE()
- VERSION()
- USER()
- PASSWORD(str)
- MD5(str);

### 聚合函数

sum()
count()
avg()

根据分组时**各自检索到**的数据行数进行求平均数

	select avg(work_hour) as avg_work_hour,em from compile_add_on_work_hour group by em
	

### 小结
	MySQL有很多内置函数，功能实用且性能高效，在空闲之余应该经常看一看MySQL的官方手册
