---
title : bash中的一些实用命令
categories : 
 - Linux 
tags :
	- Linux
---

如果我看得更远一点的话，是因为我站在巨人的肩膀上

## Part.1 实用命令

### alias

查看当前bash中设置了哪些别名

	[root@vagrant-centos65 bp]# alias
	alias cp='cp -i'
	alias l.='ls -d .* --color=auto'
	alias ll='ls -l --color=auto'
	alias ls='ls --color=auto'
	alias mv='mv -i'
	alias rm='rm -i'
	alias which='alias | /usr/bin/which --tty-only --read-alias --show-dot --show-tilde'

设置别名

先查一下，别把别人设置的别名搞没了

	[root@vagrant-centos65 bp]# type lsord
	-bash: type: lsord: not found

设置

	[root@vagrant-centos65 bp]# alias lsord='ls -lhS'
	[root@vagrant-centos65 bp]# lsord
	total 82K
	-rw-rw-r-- 1 www www  61K Jan  6 21:22 composer.lock
	drwxrwxr-x 1 www www 4.0K Dec 20 14:02 common
	...

取消别名

	[root@vagrant-centos65 bp]# unalias lsord
	[root@vagrant-centos65 bp]# lsord
	-bash: lsord: command not found

### history 查询执行命令的历史记录

好用而且简单, man history 一下

### wildcard 通配符

通俗点讲就是在bash中可以使用一些类似于正则匹配的东西，(但是语法与正则稍有不同 注意区别)

	[root@vagrant-centos65 logs]# ll
	total 332
	drwxrwxr-x 1 www www      0 Mar 18 14:42 20190315
	-rw-rw-r-- 1 www www 337315 Mar 18 16:46 app.log
	[root@vagrant-centos65 logs]# ll 2019*
	total 4
	-rw-rw-r-- 1 www www 1738 Mar 15 21:54 order.log

	[root@vagrant-centos65 logs]# ll [^0-9]*
	-rw-rw-r-- 1 www www 337315 Mar 18 16:46 app.log


## 系统中同名的命令那么多，如何选择哪一个执行

现在有这样一个问题，在linux系统中，命令同名不可避免，那么当执行一个具有多个同名文件的指令时，最终执行的指令来自哪里呢？bash对于指令的执行优先级进行了排序

1. 绝对路径
1. 别名
1. bash的内建指令
1. 去$PAHT指定的路径中寻找，取第一个

## login shell 与 nologin shell

用户具有的权限与用户的环境变量息息相关，根据用户的登录状态，shell 分为 login shell 与 nologin shell

获取到 login shell 后，环境变量的渲染顺序为

1. /etc/profile
1. 按顺序查找，只要存在一个即停止 [~ /.bash_profile|~/.bash_login|~/.profile]

获取到 login shell 后，环境变量的渲染顺序为

1. /etc/profile
1. ~/.bash_profile

结合 `su` 与 `su -`的使用即可明白上述区别

## 小结
