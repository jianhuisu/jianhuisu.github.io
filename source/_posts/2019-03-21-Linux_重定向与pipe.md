---
title : 数据流重定向与pipe
categories : 
 - Linux 
tags :
	- Linux
---

## 重定向

单个 `>` 代表覆盖 ， 双`>>` 代表追加

- stdin   < <<
- stdout  > >> 或者 1> 1>>
- stderr  2> 2>>

eg.1 正确则重定向 right.log ，错误则重定向 error.log

	[root@iZ2zeh70iv04ct6uk02dscZ machine_su]# llll 1> right.log 2> error.log
	[root@iZ2zeh70iv04ct6uk02dscZ machine_su]# cat error.log
	bash: llll: command not found

	[root@iZ2zeh70iv04ct6uk02dscZ machine_su]# ls 1> right.log 2> error.log
	[root@iZ2zeh70iv04ct6uk02dscZ machine_su]# cat right.log
	1
	2
	error.log
	foo1
	foo1big5
	foo2
	foo3
	index.html
	index.php
	right.log

eg.2 不论对错,全部重定向到 all.log

	[root@iZ2zeh70iv04ct6uk02dscZ machine_su]# ls >> all.log 2>&1
	[root@iZ2zeh70iv04ct6uk02dscZ machine_su]# cat all.log
	all.log
	error.log
	foo1
	foo1big5
	foo2
	foo3
	index.html
	index.php
	right.log

eg.3 输出到垃圾桶 `ls > /dev/null` 不占用输出

	[root@iZ2zeh70iv04ct6uk02dscZ machine_su]# ls > /dev/null
	[root@iZ2zeh70iv04ct6uk02dscZ machine_su]#

eg.4 双向重定向 一份输出、一份打印

	[root@iZ2zeh70iv04ct6uk02dscZ machine_su]# cat index.html | tee new_index.html | more
	<html>
		<div style="border:"></div>
	</html>


	[root@iZ2zeh70iv04ct6uk02dscZ machine_su]# ll
	total 28
	...
	-rw-rw-r-- 1 machine_su machine_su 45 Mar 18 14:44 index.html
	-rw-r--r-- 1 root       root       45 Mar 21 16:02 new_index.html

eg.5

	[root@iZ2zeh70iv04ct6uk02dscZ machine_su]# cat index.html
	<html>
		<div style="border:"></div>
	</html>

	[root@iZ2zeh70iv04ct6uk02dscZ machine_su]# cat index.html > new_index.html

	// 注意index.html 位置的变换  由index.html 的内容代替键盘的标准输入
	[root@iZ2zeh70iv04ct6uk02dscZ machine_su]# cat > new1_index.html < index.html

	[root@iZ2zeh70iv04ct6uk02dscZ machine_su]# ll *index*
	-rw-rw-r-- 1 machine_su machine_su 45 Mar 18 14:44 index.html
	-rw-rw-r-- 1 machine_su machine_su  1 Mar 18 14:45 index.php
	-rw-r--r-- 1 root       root       45 Mar 21 16:10 new1_index.html
	-rw-r--r-- 1 root       root       45 Mar 21 16:09 new_index.html


## pipe

管道就像浇地时候拨渠，流向这个垄的水到头以后，打开流向另外一个垄的坝堰，流过来的水都是被吸收过的了

有一个xxoo.log，预览一下，找出其中包含字母`a`的行

    [root@iZ2zeh70iv04ct6uk02dscZ machine_su]# grep a xxoo.log
	dasdfasdf
	dasdfasdf
	dasdfasdf
	aaaaaaaaa

我想统计xxoo.log中包含字母`a`的行数，如果有一个这样的文件就好了 xxoo_only_contain_a.log

	[root@iZ2zeh70iv04ct6uk02dscZ machine_su]# wc -l xxoo_only_contain_a.log

ok，可以这样做

	[root@vagrant-centos65 20190315]# grep a xxoo.log > tmp_data_file && wc -l tmp_data_file
	4

简写为

	[root@vagrant-centos65 20190315]# grep a xxoo.log | wc -l
	4

可以这样理解：

1. 当指令中包含管道符 | 时, bash 首先以 | 为分隔符对指令进行分段
1. 依次执行分段指令,并把每一段指令的标准输做临时存储(可以理解为存储到临时文件)，作为下一个命令的标准输入
1. 最后一段指令执行完成后,清空临时存储

### 常用管线命令

- sort
- cut
- uniq
- grep
- xargs  动态生成命令参数
- tr
- col
- split  文件分区
- -   标准输出的文件名
