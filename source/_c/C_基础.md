---
title: 基础知识  
---


environment

linux release centos6.5
gcc version 4.4.7 20120313 (Red Hat 4.4.7-4) (GCC)

一个c文件，首先从 main()函数开始执行

hello.c

#include <stdio.h>
void main(void)
{
	printf("hello world\n");
}

gcc -o hello hello.c && ./hello


C语言非常简洁，只有32个关键字，9种控制语句，34种运算符。
 

1 数据类型

整型
	int
	char


浮点型
	float
	double


派生数据类型
	
	pointer
	struct
	union

  	数组 array
	枚举 enum	


2 语法结构(9 种)

if else
while
do ... while
for
switch case 
break
goto
continue
return 


3 操作符

简单记忆

！ > 算术运算符 > 关系运算符 > && > || > 赋值运算符 


