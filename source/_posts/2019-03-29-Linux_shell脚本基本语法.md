---
title : shell脚本基本语法
tags :
	- Linux
---

## 注意

### shell脚本中空格很重要,会影响语义的解析

声明变量时注意`=`号左右不要有空格
	
	#!/bin/bash
	a=1       // 合法
	a = 1     // 非法

if语法结构中 if与[,[与其中内容需要使用空格隔开

	#!/bin/bash
    if [ $a = 1]  // 正确使用空格
    then
        echo 'success'
    else
        echo 'fail'
    fi

错误使用空格

	#!/bin/bash
    if[$a = 1]   
    then
        echo 'success'
    else
        echo 'fail'
    fi

### shell中判断变量是否相等时，不同类型变量比较时使用不同的操作符

shell中判断变量是否相等时，不同类型变量比较时使用不同的判读符号,在if判断中,`=`判断是否字符串是否相同,`==`也可以判断是否等于 推荐使用`==`,但是判断数字值是否相等需使用`eq` 或 `nq`。

### 函数声明时的格式问题

函数名与花括号之间至少间隔一个空格

### 一门编程语言的结构

shell也不例外 包含下面四个部分

- 变量
- 基本语法结构
- 运算符
- 函数

## 基本语法结构

### if

	if [ 条件测试 1 ]
	then
	做 1 的事情
	elif [ 条件测试 2 ]
	then
	做 2 的事情
	elif [ 条件测试 3 ]
	then
	做 3 的事情
	else
	做其他事情
	fi

### case

	case $name in
        "sujianhui")
        echo $name
        ;;

        "sujianhui1")
        echo $name
        ;;
        "sujianhui2")
        echo $name
        ;;
        "sujianhui3")
        echo $name
        ;;
        *)
        echo $name
        ;;
	esac

### for
	
	for 变量 in '值1' '值2' '值3' ... '值n'
	do
	    做某些事
	done

两个实例

循环输出 1-10

	for i in `seq 1 10`
	do
		echo $i
	done

循环输出 'dog' 、'cat' 、'pig'
	
	for animal in 'dog' 'cat' 'pig'
	do
	        echo $item;
	done

## 常用运算符

### 字符串判断运算符

两个字符串是否相等，大小写敏感

	if [ $string1 == $string2 ]
	if [ $string1 != $string2 ]
	
字符串 string 是否为空。z是 zero 的首字母，是英语「零」的意思 `-z $string`

	if [ -z $string1 ]

字符串 string 是否不为空。n是英语not的首字母，`-n $string`

	if [ -n $string1 ]

### 数字判断运算符

两个数字值是否相等,`eq`是`equal`的缩写`nq`代表`not equal`

	if [ $num1 -eq $num2 ]
	if [ $num1 -nq $num2 ]
	
数字 num1 是否小于 num2。lt 是`lower than`的缩写

	$num1 -lt $num2

数字 num1 是否小于或等于num2。`le`是`lower or equal`的缩写 

	$num1 -le $num2

数字 num1 是否大于 num2。gt 是 greater than 的缩写，是英语「大于」的意思。
	
	$num1 -gt $num2

数字 num1 是否大于或等于 num2。`ge`是`greater or equal`的缩写
	
	$num1 -ge $num2

### 文件相关运算符

`file`变量代表文件的路径

文件是否存在。e 是 exist 的首字母，表示「存在」。
	
	if [ -e $file ]

文件是否是一个目录。因为 Linux 中所有都是文件，目录也是文件的一种。d 是 directory 的首字母，表示「目录」。

	if [ -d $file ]

文件是否是一个文件。f 是 file 的首字母，表示「文件」。

	if [ -f $file ]

文件是否是一个符号链接文件。L 是 link 的首字母，表示「链接」。

	if [ -L $file ]

文件是否可读。r 是 readable 的首字母，表示「可读的」。

	if [ -r $file ]

文件是否可写。w 是 writable 的首字母，表示「可写的」。

	if [ -w $file ]

文件是否可执行。x 是 executable 的首字母，表示「可执行的」。

	if [ -x $file ]

文件 file1 是否比 file2 更新。nt 是 newer than 的缩写，表示「更新的」

	if [ $file1 -nt $file2 ]

文件 file1 是否比 file2 更旧。ot 是 older than 的缩写，表示「更旧的」。

	if [ $file1 -ot $file2 ]


## 函数

### 函数的声明
	
	function getName {
        echo $1
	}
	
### 函数的调用与传参

	setName()
	{
	        echo $1;
	}
	
	setName sujianhui

函数传参：括号内不能传参，传参方式 类似于shell脚本传参使用`$n`接收参数`$?`获取return状态值

### 变量的作用域

	#!/bin/bash
  
	self_define_func() {
	      local local_variable="I am local vaiable"
	      
	      echo "vaiiable value in function :$local_variable, not change though out has a same name va    riable"   7 }
	
	show_val(){
	      echo $1 
	}       
	
	local_variable="I am global variable"
	show_val "$local_variable"
	
	self_define_func
	show_val "$local_variable"

执行一下(当函数实参中包含空格时，传递参数使用双引号包含起来即可)

	[root@vagrant-centos65 machine_su]# ./script.sh 
	I am global variable
	vaiiable value in function :I am local vaiable, not change though out has a same name variable
	I am global variable
	
变量声明时，默认全局变量(global)，如果要在函数内部声明一个局部变量,前面添加`local`关键字修饰即可


## 参考资料

非常感谢原作者的总结

程序员联盟:[https://www.jianshu.com/p/471f8d07542e](https://www.jianshu.com/p/471f8d07542e "程序员联盟")

