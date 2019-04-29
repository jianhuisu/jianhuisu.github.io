---
title : MySQL中的运算符
categories : 
 - mysql 
tags :
	- MySQL
---

### 算数运算符

	+
	-
	*
	/,DIV          除法，求商
	%,MOD(a,b)     求模，求余数

eg.

	mysql> select 1 DIV 2;
	+---------+
	| 1 DIV 2 |
	+---------+
	|       0 |
	+---------+
	1 row in set (0.00 sec)

	mysql> select 1 DIV 1;
	+---------+
	| 1 DIV 1 |
	+---------+
	|       1 |
	+---------+
	1 row in set (0.00 sec)

求模

	mysql> select 1%2;
	+------+
	| 1%2  |
	+------+
	|    1 |
	+------+
	1 row in set (0.00 sec)

	mysql> select 3%2;
	+------+
	| 3%2  |
	+------+
	|    1 |
	+------+
	1 row in set (0.00 sec)

	mysql> select mod(3,2);
	+----------+
	| mod(3,2) |
	+----------+
	|        1 |
	+----------+
	1 row in set (0.00 sec)

注意：在求商与求模运算中，如果除数为0，那么计算结果为null。

	mysql> select 1/0;
	+------+
	| 1/0  |
	+------+
	| NULL |
	+------+
	1 row in set, 1 warning (0.02 sec)

	mysql> select 1%0;
	+------+
	| 1%0  |
	+------+
	| NULL |
	+------+
	1 row in set, 1 warning (0.00 sec)

	mysql> select 1 DIV 0;l
	+---------+
	| 1 DIV 0 |
	+---------+
	|    NULL |
	+---------+
	1 row in set, 1 warning (0.00 sec)

	mysql> select mod(1,0);
	+----------+
	| mod(1,0) |
	+----------+
	|     NULL |
	+----------+
	1 row in set, 1 warning (0.00 sec)

### 比较运算符

比较运算符可以用于比较 数字、字符串、表达式，整数当做浮点数进行比较。


	>
	<
	=
	<>	 不等于
	>=   大于等于
	<=
	<=>  安全等于，可以正确判断是否与null相等
	between ... and ...
	IN()
	LIKE "%字符%"
	IS NULL
	IS NOT NULL

对于下列操作符，如果有两个操作数中存在null，则返回结果为null，而非期望值0或者1。也就是说,不可以有`select * from user where name=null` 或者`select * from user where name='null'` 类似的用法。应该使用 `select * from user where name is null `来替代。

	>
	<
	=
	<>
	>=
	<=

对于`<=>`运算符，可以正确的对数值与null进行比较

	mysql> select 'foo'<=>null;
	+--------------+
	| 'foo'<=>null |
	+--------------+
	|            0 |
	+--------------+
	1 row in set (0.00 sec)

	mysql> select 'foo' = null;
	+--------------+
	| 'foo' = null |
	+--------------+
	|         NULL |
	+--------------+
	1 row in set (0.00 sec)

between min and max，等价于 (column >= min) and (column <= max )

IN(value1,value2,value3...) 使用易错点要注意

	mysql> select * from compare_null;
	+------+-----------+
	| id   | name      |
	+------+-----------+
	|    1 | NULL      |
	|    2 | NULL      |
	|    3 | NULL      |
	|    4 | sujianhui |
	|    4 |           |
	+------+-----------+
	5 rows in set (0.00 sec)

	mysql> select * from compare_null where id in(1,2,3,4);
	+------+-----------+
	| id   | name      |
	+------+-----------+
	|    1 | NULL      |
	|    2 | NULL      |
	|    3 | NULL      |
	|    4 | sujianhui |
	|    4 |           |
	+------+-----------+
	5 rows in set (0.00 sec)

	mysql> select * from compare_null where id in('1','2','3','4');
	+------+-----------+
	| id   | name      |
	+------+-----------+
	|    1 | NULL      |
	|    2 | NULL      |
	|    3 | NULL      |
	|    4 | sujianhui |
	|    4 |           |
	+------+-----------+
	5 rows in set (0.00 sec)

	mysql> select * from compare_null where id in('1,2,3,4');
	+------+------+
	| id   | name |
	+------+------+
	|    1 | NULL |
	+------+------+
	1 row in set, 1 warning (0.00 sec)

要使用逗号各个可能值连接起来。

	in(1,2,3,4)          正确
	in('1','2','3','4')  正确
	in('1,2,3,4')        错误 类型转化后等价于 IN(1)

regexp 正则匹配（这个很有用）

	mysql> select name from compare_null where name regexp 'su';
	+-----------+
	| name      |
	+-----------+
	| sujianhui |
	+-----------+
	1 row in set (0.03 sec)

### 逻辑运算符

与 and &&
或 or ||
非 !
异或

### 位运算符

位与  &

2 & 3 等价于  10 & 11 => bin: 01 =>dec: 2

	mysql> select 2&3;
	+-----+
	| 2&3 |
	+-----+
	|   2 |
	+-----+
	1 row in set (0.00 sec)

位或  |

2 | 3 等价于  10 | 11 => bin: 11 =>dec: 3

	mysql> select 2|3;
	+-----+
	| 2|3 |
	+-----+
	|   3 |
	+-----+
	1 row in set (0.04 sec)

位异或 ^

2 ^ 3 等价于  10 ^ 11 => bin: 01 =>dec: 1

	mysql> select 2^3;
	+-----+
	| 2^3 |
	+-----+
	|   1 |
	+-----+
	1 row in set (0.00 sec)


位取反 ~

位取反操作符的操作对象只能为一个。`~1`等价于对一位8字节int取反

	mysql> select ~1;
	+----------------------+
	| ~1                   |
	+----------------------+
	| 18446744073709551614 |
	+----------------------+
	1 row in set (0.00 sec)


位右移 >>

dec:100 => bin:1100100 。100 >> 3 等价于将100的二进制右移三位，左边补0
	result:0001100 => dec:12

	mysql> select 100>>3;
	+--------+
	| 100>>3 |
	+--------+
	|     12 |
	+--------+
	1 row in set (0.00 sec)

位左移 <<

dec:100 << 3 等价于将100的二进制左移三位，右边补0
result:1100100000 => dec:800

	mysql> select 100<<3;
	+--------+
	| 100<<3 |
	+--------+
	|    800 |
	+--------+
	1 row in set (0.00 sec)

## 小结
运算符的优先级


- 运算符的优先级比较难以记忆，可以使用小括号控制优先级，这样不论操作性与可读性都比较好。

运算符的操作项

- 注意操作项中含有`null`时的结果

顺便说一句 圣诞节快乐
