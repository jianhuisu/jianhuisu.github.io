---
title : win7利用vagrant搭建Tidb
tags : 
	- Linux
---

## 环境介绍
	
	OS  : Windows7
	CPU : i7-6700
	RAM : 16GB
	type: 64位
	disk: 1T 

## 准备工作

软件包

- virtualbox：仿真计算机的工具
- vagrant:因为virtualbox本身对自己仿真出的虚拟机操作比较麻烦，所以外面的人开发了一个虚拟机的管理工具
- centos7.box：供vagrant使用的镜像

Tips:

	1 virtualbox软件本身不要安装在C盘,安装完成之后,务必在全局设置中将虚拟机实例默认存储目录切换到C盘外
	2 将vagrant安装镜像存储位置设置到C盘外

## Step.1 创建第一个实例

	D:\www>develop_host>vagrant box add ct7_node1 develop_environment.box
	D:\www>develop_host>vagrant init

init后，当前目录生成一个Vagrantfile，设置网络连接为桥接模式，这样该虚拟机独占一个IP

	Vagrant.configure("2") do |config|
	 
	  config.vm.box = "ct7_node1"
	  config.ssh.username = "vagrant"
	  config.ssh.password = "vagrant"
	  config.ssh.insert_key = false
	  config.vm.network "public_network"	  
	  
	end

配置完成后,启动该实例

	D:\www>develop_host>vagrant up

启动完成后，登入虚拟机
	
	D:\www>develop_host>vagrant ssh
	... 安装一些乱七八糟的软件

查看实例状态

	D:\www\ct7_node1>vagrant box list
	==> vagrant: A new version of Vagrant is available: 2.2.4!
	==> vagrant: To upgrade visit: https://www.vagrantup.com/downloads.html
	
	centos7     (virtualbox, 0)
	ct6.5       (virtualbox, 0)
	ct7_node1   (virtualbox, 0)
	ct7_node2   (virtualbox, 0)
	develop_env (virtualbox, 0)
	
	D:\www\ct7_node1>vagrant global-status
	id       name    provider   state    directory
	-------------------------------------------------------------------------
	918fd20  default virtualbox running  D:/www/centos
	119806d  default virtualbox running  D:/www/ct7_node1
	fe0d6eb  default virtualbox poweroff D:/www/ct7_node2
	
	The above shows information about all known Vagrant environments
	on this machine. This data is cached and may not be completely
	up-to-date. To interact with any of the machines, you can go to
	that directory and run Vagrant, or you can use the ID directly
	with Vagrant commands from any directory. For example:
	"vagrant destroy 1a2b3c4d"

停止该虚拟机
	
	D:\www>develop_host>vagrant halt
	
导出该虚拟机，首先去virtualbox安装目录下查找该实例的注册ID

	D:\Program Files\Oracle\VirtualBox>vboxmanage list vms
	"centos_default_1556089212841_1875" {dfe13dde-d41e-4f7d-9025-ea788b658b94}
	"ct7_node1_default_1556090991911_25320" {2410ca93-bc49-43bd-a39b-f877a529cbd9}
	"ct7_node2_default_1556091574788_4422" {ad55a355-0904-43cc-8c64-0685b41e2daf}
	
根据拿到的ID，进行box导出

	D:\www>develop_host>vagrant package --base ct7_node1_default_1556090991911_25320 --output ./ct7node.box

使用该镜像，多创建几个实例...
	
	D:\www>node2>vagrant box add node2 ./../develop_host/ct7node.box
	...
	

## Step.2 虚拟机中安装 Mysql 



## 参考地址

docker与vagrant 区别：https://www.cnblogs.com/vikings-blog/p/3973265.html





