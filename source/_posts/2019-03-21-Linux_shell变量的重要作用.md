---
title : shell变量的重要作用
tags : 
	- Linux
---

## shell变量的分类

shell变量分为两类

- 自定义变量
- 环境变量

显示所有环境变量

	[root@vagrant-centos65 ~]# env | wc -l
	23

显示所有变量

	[root@vagrant-centos65 ~]# set | wc -l
	58

PSI

## 如何控制shell变量

### 查询

	[root@vagrant-centos65 ~]# echo $PATH
	/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/usr/local/php/bin:/root/bin

或者(推荐)
	
	[root@vagrant-centos65 ~]# echo ${PATH}  
	/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/usr/local/php/bin:/root/bin

### 声明

变量名只能包含数字、字母、下划线

	[root@vagrant-centos65 ~]# my_define_variable=xxoo
	[root@vagrant-centos65 ~]# echo $my_define_variable
	xxoo
	
一般情况下 全大写变量为系统默认变量

#### 变量名不能以数字开头

	[root@vagrant-centos65 ~]# 1my_define=xxoo
	-bash: 1my_define=xxoo: command not found

#### 等号两边不能存在空格

	[root@vagrant-centos65 ~]# my_define = xxoo
	-bash: my_define: command not found

#### 当变量内容包含空格时，需要使用单引号或者双引号将值包裹起来，赋值时，单引号不能解析值中的shell变量，双引号可以解析
	
	[root@vagrant-centos65 ~]# echo my_name1
	my_name1
	[root@vagrant-centos65 ~]# my_name2='$my_name1'
	[root@vagrant-centos65 ~]# echo $my_name2
	$my_name1
	[root@vagrant-centos65 ~]# my_name3="$my_name1"
	[root@vagrant-centos65 ~]# echo $my_name3
	my name is

#### 可以使用反斜杠对特殊字符进行转义

#### 使用反引号`cmd`或者$(cmd)将命令的输出赋值到变量

 
	[root@vagrant-centos65 ~]# my_name4=`pwd`
	[root@vagrant-centos65 ~]# echo $my_name4
	/root
	[root@vagrant-centos65 ~]# my_name5=$(pwd)  // 推荐使用这种方式
	[root@vagrant-centos65 ~]# echo $my_name5
	/root

#### 使用分号:操作变量内容累加

	[root@vagrant-centos65 ~]# my_name5=${my_name5}:dfdfdfdfdfdfdfdf
	[root@vagrant-centos65 ~]# echo $my_name5
	/root:dfdfdfdfdfdfdfdf

Tips:默认情况下,shell变量的值类型全部为字符串类型，将两个字符型shell变量代入到计算式中，计算式不会生效

	[root@vagrant-centos65 ~]# num1=1
	[root@vagrant-centos65 ~]# num2=2
	[root@vagrant-centos65 ~]# num3="$num1+$num2"
	[root@vagrant-centos65 ~]# echo $num3
	1+2

使用`declare -i`命令可以在变量定义时限定变量的类型

对已经赋值完成的变量进行类型声明无效

	[root@vagrant-centos65 ~]# declare -i num3   
	[root@vagrant-centos65 ~]# echo $num3
	1+2

在赋值时声明
  
	[root@vagrant-centos65 ~]# declare -i num3="$num1+num2"
	[root@vagrant-centos65 ~]# echo $num3
	3

#### 数组变量的声明 

	[root@vagrant-centos65 ~]# var[1]=1111
	[root@vagrant-centos65 ~]# echo $var
	
	[root@vagrant-centos65 ~]# echo $var[1]
	[1]
	[root@vagrant-centos65 ~]# echo ${var[1]}
	1111

#### 从键盘读取输入并赋值到一个变量

这个东东很重要，是与用户交互的重要渠道

	[root@vagrant-centos65 bp]# read clos_message
	dddddddddddddddddddddd
	[root@vagrant-centos65 bp]# echo $clos_message
	dddddddddddddddddddddd

### 更新

重新赋值即可

### 销毁

	[root@vagrant-centos65 ~]# echo $my_name5
	/root:dfdfdfdfdfdfdfdf
	[root@vagrant-centos65 ~]# unset my_name5
	[root@vagrant-centos65 ~]# echo $my_name5
	
	[root@vagrant-centos65 ~]# 


### 作用域

自定义变量不能被子程序使用，环境变量可以被子程序使用

	[root@vagrant-centos65 ~]# echo $my_name3
	my name is

在当前bash下下达的任何指令，都会以该bash进程的子程序身份去执行。

对已存在的自定义变量使用 export 升级为环境变量	

	[root@vagrant-centos65 ~]# export my_name3
	[root@vagrant-centos65 ~]# env | grep my_name3
	my_name3=my name is

Tips:为什么环境变量可以被子程序引用呢

内存引用的机制

### 特殊却有用的变量	

命令提示符 ： PS1

	[root@vagrant-centos65 ~]# echo $PS1
	[\u@\h \W]\$

当前shell进程ID
	[root@vagrant-centos65 ~]# echo $$
	3480

命令执行回传码(一般上一个命令执行成功，该变量值为0,上一个命令执行失败，该变量为非0数字)

	[root@vagrant-centos65 ~]# echo $?
	0
	[root@vagrant-centos65 ~]# lll
	-bash: lll: command not found
	[root@vagrant-centos65 ~]# echo $?
	127

看来以后使用php写后台应用，任务执行成功后要`exit(0)`了

	[root@vagrant-centos65 ~]# php -r "echo 'xxoo';exit(1);"
	xxoo
	[root@vagrant-centos65 ~]# echo $?
	1
	[root@vagrant-centos65 ~]# php -r "echo 'xxoo';exit(0);"
	xxoo
	[root@vagrant-centos65 ~]# echo $?
	0


这个命令执行回传码很有用，有时候处于业务的需要，我们需要同时执行两个命令


### 变量的测试、替换

#### 如何测试一个变量是否存在、或者是否为空呢

`-` 的用户类似于php的`isset()`

	[root@vagrant-centos65 bp]# echo ${myname-sujianhui}
	sujianhui
	[root@vagrant-centos65 bp]# echo $myname
	
	[root@vagrant-centos65 bp]# myname=""
	[root@vagrant-centos65 bp]# echo ${myname-sujianhui}
	
	[root@vagrant-centos65 bp]# 

等价于
	
	<?php
		$c = $myname ?? 'sujianhui'; // => $c = isset($myname) ? $myname : 'sujianhui';
		echo $c;

`:-` 的用户类似于php的`empty()`

	[root@vagrant-centos65 bp]# myname=""
	[root@vagrant-centos65 bp]# echo ${myname:-sujianhui}
	sujianhui
	[root@vagrant-centos65 bp]# 

等价于

	<?php
		$c = $myname ?: 'sujianhui';
		echo $c;

#### 如何对变量的内容进行操作

todo 具体用法去查询鸟哥私房菜对应章节说明，我现在用的不算多，用熟了再来更新