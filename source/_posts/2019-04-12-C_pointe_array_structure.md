---
title : 指针、数组、结构易错点整理
categories : 
 - C 
tags :
	- C
---

## 指针

`*` 操作符的三种用法

### Usage.1 指针变量的声明

声明变量时，在变量类型与变量名之间使用，表示声明的变量为指针变量`int * p1`、`int* p2`、`int *p3`

	void main(void)
	{
	    int * p1;
	    int* p2;
	    int *p3;

	    int a = 1;

	    p1 = &a;
	    p2 = &a;
	    p3 = &a;

	    printf("pointer p1 values %p\n",p1);
	    printf("pointer p2 values %p\n",p2);
	    printf("pointer p3 values %p\n",p3);
	    printf("pointer p3 point values is %d\n",*  p3);
	    printf("int values %d\n",a);
	}

使用注意事项：

- 注意指针类型，*、指针变量名之间的空格
- 指针变量的值为无符号整数，但是不可以对指针变量使用一般整数运算公式 实际上指针变量的值是一种新的数据类型 使用%p转换符打印
- 声明指针变量时必须指定指针变量的类型

Tips:为什么指针变量有要有类型 ?

指针变量的值为一个地址(例如:0x7fff60a2d05c),该地址指定变量值在内存段中的开始位置,指针变量的类型指定该内存段的结束位置。变量的类型大小是固定的,比如一般int(可以使用sizeof获取)类型占据4个字节 4*8 = 32bit,那么根据起始地址与变量值所占位数,可以得出数据段的结束位置。

        0x7fff60a2d05c + 0x20 = 0x7fff60a2d05c

概括为: 指针变量的值决定从哪取,指针变量的类型决定取多长

### Usage.2 解引用

对已经初始化的指针变量进行解引用,返回指针变量指向地址上的数据内容

	int a = 1;
	int *p3;
	p3 = &a;

则`*p3`的返回值为 1。

Tips:千万不可以解引用未初始化的指针

    int *p;
    *p = 5;

可能造成后果:

- 擦写数据
- 程序崩溃


### Usage.3 乘法

计算表达式中，表示数学中的乘法

        int a;
        a = 512 * 512;


### 指向函数的指针



## 数组

!数组名与数组首元素地址相同

### 数组声明

`*`、`()`、`[]` 的优先级

- 数组名后面的 [] 与函数名后面的() 优先级相同 并高于 *
- [] () 优先级相同，从左向右结合

常见的复杂声明

    int * uuf[3]

[] 的优先级高于 * ,首先这个一个包含 3 个元素的数组 ,然后确定数组元素的类型 : 指向 int 的指针
这是一个包含三个指针元素的指针数组

    int (* uuf)[3]

从左向右结合  * uuf 先结合,这是一个指针 (不理解)

    int sex[12][50]

12 个包含 50个 int元素数组 组成的二维数组

    int * uuf[3][4]

3*4 的二维数组 元素为int 型指针

    int (* uuf)[3][4]

一个指向 3*4 二维数组的指针

### 二维数组
### 指针数组

## 数据结构

### structure 结构的声明

typedef 并没有创建新的类型 只不过是为现有类型创建了简写

	#include <stdio.h>
	#include <stdlib.h>
	#include <string.h>

	struct user{
	    int id;
	    char name[40];
	};

	typedef struct user_alias{
	    int id;
	    char name[40];
	}USERALIAS;

	typedef char BYTE;

	void main(void)
	{
	    struct user employee;
	    USERALIAS employee1;
	    BYTE name[10] = "my name";

	    employee.id   = 10010;
	    strcpy(employee.name,"sjh");
	    printf("today entry employee id is %d and name is %s \n",employee.id,employee.name);

	    scanf("%s %d",&employee1.name,&employee1.id);
	    printf("today entry employee1 id is %d and name is %s \n",employee1.id,employee1.name);

	    printf("typedef  %s \n",name);
	}

执行结果

	[root@vagrant-centos65 structure]$>t struct_user.c
	today entry employee id is 10010 and name is sjh
	dfdfdf 444  // 此处为输入值
	today entry employee1 id is 444 and name is dfdfdf
	typedef  my name

### 指向结构变量的指针

him 是一个指向结构的指针,him->face 代表该指针所指向结构的一个成员。如果`library`是一个结构变量,那么需要使用`&library`获取结构变量地址

	struct book library;
	struct book * him;
	him = &library

`*him`等价于`library`,`library.face` 等价于 `(*him).face`

### 将结构变量当做函数的参数

将结构变量当做参数传递给函数时，我们有两种方式可以选择,传递给函数指针，或者传递给函数结构变量。

传递指针:

- 优点: 直接操作原始数据,不用生成数据副本 效率高
- 缺点：可读性较差，无法保护原始数据 (可以使用 const 限定符禁止修改)

传递结构变量:

- 优点: 可读性高,操作对象为数据副本 原始数据安全
- 缺点: (自动变量)生成数据副本,消耗时间、空间

总结: 数据规模小时采用传递结构变量，数据规模较大时传递结构指针

### 结构变量的动态赋值

	type *p;
	if(NULL == (p = (type*)malloc(sizeof(type))))
	{
	    perror("error...");
	    exit(1);
	}

	...

	free(p);
	p = NULL;/*请加上这句*/
