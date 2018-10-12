---
layout:     post
title:      PHP编程经验
subtitle:   declare语句结构
date:       2018-10-11
author:     machine_su
header-img: img/post-bg-kuaidi.jpg
catalog: true
tags:
    - PHP
---

今天发现php一个结构挺有意思
	
	function declare_handler()
	{
	    echo "called \n";
	}
	
	register_tick_function('declare_handler');
	declare(ticks=1){
	
	    for($i = 0;$i<2;$i++)
	    {
	        $a = $i;
	    }
	
	}
	
	for($i = 0;$i<2;$i++)
	{
	    $a = $i;
	}

执行结果

	[root@vagrant-centos65 frame]# php declare.php
	called 
	called 
	called 

`declare(ticks=N)` 结构会在结构中 代码代买片段每执行 `N` 行后调用一次 declare_handler()。下面是官方给的实例

	<?php
	// these are the same:
	
	// you can use this:
	declare(ticks=1) {
	    // entire script here
	}
	
	// or you can use this:
	declare(ticks=1);
	// entire script here
	?>

#### 参考资料 ####
[韩天峰pcntl博客](http://rango.swoole.com/archives/364 "swoole")
[官方手册](http://www.php.net/manual/zh/control-structures.declare.php "php")