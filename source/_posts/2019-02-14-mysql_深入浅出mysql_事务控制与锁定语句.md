---
title : 事务控制与锁定语句
tags :
	- MySQL
---

## 事务控制

1. start transaction | begin 
1. 向支持事务的表中插入数据 
1. commit/rollback;

	mysql> start transaction;
	Query OK, 0 rows affected (0.00 sec)
	
	mysql> insert into user(id,name) values(10,'dddd');
	Query OK, 1 row affected (0.00 sec)
	
	mysql> commit;
	Query OK, 0 rows affected (0.03 sec)


chain：在事务结束以后立刻开启一个新的与刚结束事务的隔离级别一致的新事务

	mysql> begin;
	Query OK, 0 rows affected (0.00 sec)
	
	mysql> insert into user(id,name) values(11,'dddd');
	Query OK, 1 row affected (0.00 sec)
	
	mysql> commit;
	Query OK, 0 rows affected (0.02 sec)
	
	mysql> begin;
	Query OK, 0 rows affected (0.00 sec)
	
	mysql> insert into user(id,name) values(12,'1212121212');
	Query OK, 1 row affected (0.00 sec)
	
	mysql> commit and chain;
	Query OK, 0 rows affected (0.13 sec)




备注：

1 如果在锁表期间开启了一个事务，那么原来的锁会被隐式的释放
	
	session1                                    session2


	mysql> lock table user write;
	Query OK, 0 rows affected (0.00 sec)

												mysql> select * from user;
													waiting........
	
	mysql> start transaction;
	Query OK, 0 rows affected (0.00 sec)
												
											    +----+---------------+
												| id | name          |
												+----+---------------+
												|  1 | ddd           |
												| 12 | 1212121212    |
												| 13 | 1212121212    |
												+----+---------------+
		
2 savepoint  可以将事务按顺序设置锚点，回滚到指定锚点
	
	mysql> begin;
	Query OK, 0 rows affected (0.00 sec)
	
	mysql> insert into user(id,name) values(15,'savepoint1');
	Query OK, 1 row affected (0.00 sec)
	
	mysql> savepoint save1;
	Query OK, 0 rows affected (0.05 sec)
	
	mysql> insert into user(id,name) values(16,'savepoint2');
	Query OK, 1 row affected (0.00 sec)
	
	mysql> rollback to savepoint save1;
	Query OK, 0 rows affected (0.00 sec)
	
	mysql> select * from user;
	+----+---------------+
	| id | name          |
	+----+---------------+
	|  1 | ddd           |
	| 10 | dddd          |
	| 11 | dddd          |
	| 12 | 1212121212    |
	| 13 | 1212121212    |
	| 15 | savepoint1    |
	+----+---------------+
	14 rows in set (0.00 sec)
	
	mysql> commit;
	Query OK, 0 rows affected (0.09 sec)

																	

## 锁定语句

锁的拥有者为当前线程，线程断开后，锁隐式释放
读为共享锁，写为独占锁
读跟写为串行操作

	lock table user read;
	unlock tables;
	


扩展：

InnoDB支持分布式事务，但是崩溃后，数据恢复会存在一定问题