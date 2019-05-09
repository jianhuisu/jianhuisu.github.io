---
title : "CGI、Fast-cgi、php-fpm、php-cgi区别"
categories : 
 - php 
tag :
	- PHP
---

WebServer可以直接返回静态资源，但是对于资源的其它操作，例如计算、压缩等WebServer需要借助其它应用程序，
例如我们常用的工具语言：PHP、C、JAVA等等，但是语言层出不穷，要实现WebServer与各种语言无缝连接，需要双方遵守共同的规范

## CGI 通用网关接口

CGI（Common Gateway Interface）为上述协议的初始版本，协议规定，CGI协议可以使用任何一种语言实现，只要这种语言具有stdin、stdout以及环境变量。
初始版本的协议落地后实现为 fork->and->execute,PHP为例：

1. 浏览器发送动态请求到服务器，服务器通知PHP解释器进行加载(接收标准输入、环境变量，解析php.ini)
1. PHP解释器fork一个子进程php-cgi执行动态脚本
1. php-cgi执行完毕后将标准输出返回至服务器，该子进程被销毁
1. 服务器将输出结果集返回至浏览器


## Fast-CGI 快速通用网关接口

Fast-CGI (Fast Common Gateway Interface) 是CGI协议的改良版本。在CGI协议的实现中，对于每一个请求,PHP都必须重新解析php.ini、重新载入全部扩展并重初始化全部数据结构。在Fast-CGI协议的实现中，PHP采用了Master-Worker的工作模型，通过Master进程对Worker进程进行分配调度。这套工作模型实现就是我们常提的：PHP-FPM(FastCGI Process Manager：FastCGI进程管理器，5.3 之后集成在php包中)

	[root@vagrant-centos65 sll]# ps -ef | grep php
	root      1092     1  0 09:45 ?        00:00:01 php-fpm: master process (/usr/local/php/etc/php-fpm.conf)
	www       1093  1092  0 09:45 ?        00:00:00 php-fpm: pool www
	www       1094  1092  0 09:45 ?        00:00:00 php-fpm: pool www
	root      3859  3498  0 17:15 pts/1    00:00:00 grep php

此时，该协议的执行过程为

1. Web Server启动时载入FastCGI进程管理器
1. php-fpm初始化，启动多个php-cgi解释器进程并等待来自Web Server的连接
1. 当客户端请求到达Web Server时，php-fpm进程管理器选择并连接到一个CGI解释器。Web server将CGI环境变量和标准输入发送到FastCGI子进程php-cgi。
1. FastCGI子进程完成处理后将标准输出和错误信息从同一连接返回Web Server。当FastCGI子进程关闭连接时，请求便告处理完成。FastCGI子进程接着等待并处理来自php-fpm的下一个连接。在CGI模式中，php-cgi在此便退出了。

## Nginx与php-fpm是如何进行通讯的

因为一个web请求是由nginx转发到php-fpm,所以可以称`nginx`是`php-fpm`的上游服务,`php-fpm`是`nginx`的下游服务。两者之间的通讯方式主要有两种

- TCP调用  通过TCP协议进行信息通讯
- UNIX SOCKET 通过socket文件进行通讯

1 tcp 方式主要配置

nginx 

	server {
		...
		
		location ~ \.php$ {
            root           html;
            fastcgi_pass   127.0.0.1:9000; // 将标准输入通过tcp协议转发至下游
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }

		...
	}

php-fpm 

	[root@vagrant-centos65 php-fpm.d]$>egrep -v '(;|^$)' /usr/local/php7/etc/php-fpm.d/www.conf
	[www]
	user = www
	group = www
	listen = 127.0.0.1:9000 // 通过tcp协议接收上游的标准输入
	...

unix socket 方式

nginx 

	server {
		...
		
		location ~ \.php$ {
            root           html;
            fastcgi_pass   unix:/dev/shm/fpm-cgi.sock; // nginx将标准输入写入到下游的socket文件, 或者从socket读取php-fpm的标准输出
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }

		...
	}

php-fpm 

	[root@vagrant-centos65 php-fpm.d]$>egrep -v '(;|^$)' /usr/local/php7/etc/php-fpm.d/www.conf
	[www]
	user = www
	group = www
	listen = /dev/shm/fpm-cgi.sock // 从socket文件读取上游的标准输入(nginx写入到socket中的信息) 或者将标准输出写入到socket
	...

注意:这种模式下nginx进程与php-fpm进程都需要具有对`php-cgi.socket`文件具有读写权限
		
理论上讲socket不需要经过路由,会比tcp方式快,但是根据实际使用情况的反馈,socket方式`502`的异常率要高于tcp。
我没有实操比对过两者的性能、稳定性的具体差异。平时工作多以tcp方式为主,tcp很稳定,而且做负载均衡也很方便。

## 小结

CGI是webserver与语言应用进行通讯的协议，Fast-CGI是CGI的改良版本，PHP-FPM是PHP对Fast-CGI的实现。


