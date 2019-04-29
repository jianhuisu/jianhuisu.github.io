---
title : 什么是bash
categories : 
 - Linux 
tags :
	- Linux
---

## 什么是shell

可以与内核通讯的接口

## 什么是bash

bash是shell的一种，如何查看当前 linux distribution 支持哪些shell呢

	[root@iZ2zeh70iv04ct6uk02dscZ www]# cat /etc/shells
	/bin/sh
	/bin/bash
	/sbin/nologin
	/usr/bin/sh
	/usr/bin/bash
	/usr/sbin/nologin

不同类型的shell具有不同的命令解析规则，不同的内置工具。在bash这个shell中，可以使用type命令判断一个命令是bash内置命令还是用户后来安装的应用

	[root@iZ2zeh70iv04ct6uk02dscZ www]# type ls
	ls is aliased to `ls --color=auto'
	[root@iZ2zeh70iv04ct6uk02dscZ www]# type php
	php is /usr/local/php/bin/php  // 用户自定义
	[root@iZ2zeh70iv04ct6uk02dscZ www]# type source
	source is a shell builtin  // bash 内置命令



## 我用的是哪个shell

### question.1 如何查看自己使用的是哪个shell

	[root@iZ2zeh70iv04ct6uk02dscZ www]# env | grep SHELL
	SHELL=/bin/bash

### question.2 为什么我使用的是这个shell

上帝创造你的时候就已经注定

	[root@iZ2zeh70iv04ct6uk02dscZ www]# grep -e 'root\|admin\|machine_su\|ftp_user' /etc/passwd
	root:x:0:0:root:/root:/bin/bash
	operator:x:11:0:operator:/root:/sbin/nologin
	admin:x:1000:1000::/home/admin:/bin/bash
	machine_su:x:1002:1002::/home/machine_su:/bin/bash
	ftp_user:x:1003:1003::/home/ftp_user:/bin/bash

观察看上述用户信息的末尾已经限定用户使用的shell类型

### question.3 如何换个shell用用

求一求root哥，给改一下passwd

	[root@iZ2zeh70iv04ct6uk02dscZ www]# vim /etc/passwd

	machine_su:x:1002:1002::/home/machine_su:/bin/sh
	:wq

	su machine_su
	sh-4.2$ echo $SHELL
	/bin/sh

## bash shell 的实用技巧

如果我写了一长串shell，然后发现中间写错了，那么如何删除这一长串命令呢

- [ctrl+ u] 删除光标到指令串最前端部分
- [ctrl+ k] 删除光标到指令串最末尾部分
- [ctrl+ a] 移动光标到行首
- [ctrl+ e] 移动光标到行尾
- [ctrl+ c] 清空整个指令串
- cmd1 \[enter] cmd2   分行显示指令串
