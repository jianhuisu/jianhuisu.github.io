---
title : PHPSTORM+XDEBUG远程断点调试原理
tags : 
	- PHP
---

xdebug 是php的一个扩展，提供断点调试以及性能分析等功能。常配合phpstorm作为调试工具。

## 服务器(代码运行端 Linux系统)

xdebug扩展在php.ini中有如下配置

	[root@vagrant-centos65 ~]# php -i | grep xdebug\.remote*
	xdebug.remote_addr_header => no value => no value
	xdebug.remote_autostart => Off => Off
	xdebug.remote_connect_back => Off => Off
	xdebug.remote_cookie_expire_time => 3600 => 3600
	xdebug.remote_enable => On => On
	xdebug.remote_log => no value => no value
	xdebug.remote_mode => req => req
	xdebug.remote_timeout => 200 => 200
	xdebug.remote_handler => dbgp => dbgp
	xdebug.remote_host => 172.16.125.153 => 172.16.125.153
	xdebug.remote_port => 9001 => 9001

- dbgp ： dbgp是一个简单的协议，用于调试应用程序的语言工具和引擎
- remote_host： phpstorm 所在主机的IP
- remote_port： phpstorm 所在主机用于调试信息发送与接收的端口	

## 开发机端(phpstorm所在机器 Windows系统)：

在菜单`phpstrom setting > languages & frameworks > php > debug` 中设置 `debug_port`。该端口值配置成功后，每次点击phpstorm右上角的`监听` 按钮时，phpstrom启动进程监听`9001`，用于发送断点调试的步进信息，接收此时程序中的调试信息用于phpstorm调试栏回显

	Debug port : 9001  // 与服务器php.ini中配置的`xdebug.remote_port`值保持一致


在菜单`phpstrom setting > languages & frameworks > php > debug > DBGp proxy` 中设置 

	IDE key : PHPSTORM
	Host    : 172.16.125.253
	Port    : 80

  
## 运行

浏览器中输入

	http://hostname/index.php?XDEBUG_SESSION_START=PHPSTROM

如果在浏览器安装了调试插件。简单点下按钮就开启调试器。当这些插件激活会直接设置XDEBUG_SESSION的cookie值，代替XDEBUG_SESSION_START。

当服务端php-fpm在标准输入中解析到`XDEBUG_SESSION_START=PHPSTROM`时，会与php.ini中的xdebug远程调试主机的预注册信息进行比对，若信息吻合，则php-fpm启动一个进程与phpstorm所在主机的`xdebug.remote_port`端口，也就是`9001`端口进行通信。当我们再phpstorm中开启调试模式后,通过对端口占用的查看，可以验证是phpstorm在使用9001端口通信

	D:\www\centos>netstat -ano | find ":9001"
    TCP    0.0.0.0:9001           0.0.0.0:0              LISTENING       6212
    TCP    172.16.125.153:9001    172.16.125.253:49779   ESTABLISHED     6212

	D:\www\centos>tasklist | find "6212"
	PhpStorm.exe                  6212 Console                    1     61,804 K

此时在服务端(Linux)对端口占用进行追溯，可以发现，是php-fpm进程再与远端phpstorm进行通信

	[root@vagrant-centos65 ~]# lsof -i:49779
	COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
	php-fpm 1091  www    4u  IPv4  18978      0t0  TCP 172.16.125.253:49779->172.16.125.153:etlservicemgr (ESTABLISHED)


以上是描述是CGI模式下的通讯原理，但是如果是CLI模式下，如何触发调试呢

vim php.ini 

	[xdebug]
    ...
    xdebug.remote_enable = on
    xdebug.remote_host = 172.16.125.153
    xdebug.remote_port = 9001
    xdebug.remote_autostart = on  // 设置为自动打开 意为自动向远端断点调试器发送调试信息

重启php-fpm，然后在phpstorm中启动监听调试信息按钮。

运行php脚本，即可发现`qq.php`在编辑器中断点处停止

	php qq.php