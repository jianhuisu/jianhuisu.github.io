---
title : mysql最大连接数
tags :
	- mysql
	- PHP
---


## 什么是mysql的最大连接数

### mysql 的几个比较有用的命令
	
显示当前进程列表

	show full processlist
显示配置文件中`max_connections`（允许的最大并发连接数）

	show variables like 'max_connections';

显示mysql记录中曾达到的最大并发连接数

	show global status like 'Max_used_connections';

显示进程信息

	show status like 'Threads%';

	+-------------------+-------+
	| Variable_name     | Value |
	+-------------------+-------+
	| Threads_cached    | 0     |
	| Threads_connected | 4     |  等价于 上文 show full processlist 的结果
	| Threads_created   | 4     |
	| Threads_running   | 2     |  这个数值一般远低于connected数值
	+-------------------+-------+

执行 connect.php

	<?php	

	$conn = mysqli_connect("127.0.0.1","root","MyName@2991","qq");

    if (!$conn) {
        echo "连接失败！";
        echo mysqli_connect_error();
        exit();
    }


    $conn1 = mysqli_connect("127.0.0.1","root","MyName@2991","qq");

    if (!$conn1) {
        echo "连接失败！";
        echo mysqli_connect_error();
        exit();
    }

    $conn2 = mysqli_connect("127.0.0.1","root","MyName@2991","qq");

    if (!$conn2) {
        echo "连接失败！";
        echo mysqli_connect_error();
        exit();
    }

    mysqli_close($conn2);

    while(1){

    }


在mysql的命令面板中执行 `show full processlist` 

	+----+-----------------+----------------------+------+---------+-------+------------------------+-----------------------+
	| Id | User            | Host                 | db   | Command | Time  | State                  | Info                  |
	+----+-----------------+----------------------+------+---------+-------+------------------------+-----------------------+
	|  4 | event_scheduler | localhost            | NULL | Daemon  | 27862 | Waiting on empty queue | NULL                  |
	|  9 | root            | 172.16.125.191:52866 | qq   | Sleep   |    16 |                        | NULL                  |
	| 32 | root            | localhost            | NULL | Query   |     0 | starting               | show full processlist |
	| 33 | root            | localhost:39304      | qq   | Sleep   |   520 |                        | NULL                  |
	| 34 | root            | localhost:39305      | qq   | Sleep   |   520 |                        | NULL                  |
	+----+-----------------+----------------------+------+---------+-------+------------------------+-----------------------+

以上结果可以观察到：
	
- 在一个php进程中可以可以通过调用`mysqli_connect()`函数创建**一个或者多个mysql连接**
- 除非手动销毁mysql连接，否则该**连接存活于该php进程的整个生命周期**，直至php脚本结束而异常关闭

程序执行完毕之后，连接不是通过Connection.close()关闭的，而是由于程序执行完毕，导致进程终止，造成与数据库的连接异常关闭，所以最后会出现TCP的RST报文。

	tcpdump -i any port 3306
	19:00:19.449790 IP localhost.39309 > localhost.mysql: Flags [F.], seq 110, ack 90, win 513, options [nop,nop,TS val 28139132 ecr 28119234], length 0
	19:00:19.449927 IP localhost.mysql > localhost.39309: Flags [F.], seq 90, ack 111, win 512, options [nop,nop,TS val 28139132 ecr 28139132], length 0
	19:00:19.449934 IP localhost.39309 > localhost.mysql: Flags [.], ack 91, win 513, options [nop,nop,TS val 28139132 ecr 28139132], length 0
	19:00:19.450026 IP localhost.39310 > localhost.mysql: Flags [F.], seq 110, ack 90, win 513, options [nop,nop,TS val 28139132 ecr 28119235], length 0
	19:00:19.450090 IP localhost.mysql > localhost.39310: Flags [F.], seq 90, ack 111, win 512, options [nop,nop,TS val 28139132 ecr 28139132], length 0
	19:00:19.450093 IP localhost.39310 > localhost.mysql: Flags [.], ack 91, win 513, options [nop,nop,TS val 28139132 ecr 28139132], length 0

异常关闭连接数由原来的12增加为14

	show status like '%Aborted_clients%';
	+------------------------+-------+
	| Variable_name          | Value |
	+------------------------+-------+
	| Aborted_clients        | 14    |
	| Mysqlx_aborted_clients | 0     |
	+------------------------+-------+

### 如何合理的设置mysql最大连接数
	
mysql默认设置max_connections值为 100 ，最大为16384，max_connections值并不是设置的越大越好，首先要考虑到服务器可以支撑的最大连接数，其次考虑实际情况下，mysql服务器
曾经达到过的并发数峰值的最大值。一般业内推荐 Max_used_connections / max_connections * 100% ≈ 85% 方案。如果发现比例在10%以下，MySQL服务器连接上限就设置得过高了。


## 为什么连接mysql耗时

### php连接mysql的过程

MySQL的通信协议是基于TCP传输协议的，而且该协议是二进制协议，不是类似于HTTP的文本协议，其中建立连接的过程具体如下：

第1步：建立TCP连接，通过三次握手实现；
第2步：客户端发送认证包，用于用户验证，验证成功后，服务器返回OK响应，之后开始执行命令；
用户验证成功之后，会进行一些连接变量的设置，比如字符集、是否自动提交事务等，其间会有多次数据的交互。完成了这些步骤后，才可以进行执行进行真正的命令操作

举个例子：
- 
- 你追到了女神，约女神出去吃午饭，接着吃晚饭，接着吃早饭...吃了N顿饭以后，有一天你不想跟女神吃饭了，你跟女神分手。// 这是一次连接，多次查询 
- 你追到了女神，约女神出去吃午饭，吃完饭就分手了。晚上你又想约女神吃饭，你还需要在重新追她一遍，你不嫌烦吗。// 这相当于建立一次连接后，只执行了一次query就关闭，query语句的耗时可能还没有连接建立的耗时大）。
- 追一个女神就够了，找两个的话，会影响写代码。// 这就是php的数据库操作类要要使用单例模式的原因

### mysql的进程与线程

待完善...
个事务可能会产生一个或多个线程，

## 参考资料

[https://zhidao.baidu.com/question/1643300559495813620.html](https://zhidao.baidu.com/question/1643300559495813620.html "https://zhidao.baidu.com/question/1643300559495813620.html")
[https://www.cnblogs.com/haciont/p/6277675.html](https://www.cnblogs.com/haciont/p/6277675.html "https://www.cnblogs.com/haciont/p/6277675.html")
[https://blog.csdn.net/lmy86263/article/details/76165714](https://blog.csdn.net/lmy86263/article/details/76165714 "https://blog.csdn.net/lmy86263/article/details/76165714")
[https://blog.csdn.net/XingKong22star/article/details/48730823](https://blog.csdn.net/XingKong22star/article/details/48730823 "https://blog.csdn.net/XingKong22star/article/details/48730823")
[https://blog.csdn.net/AlbertFly/article/details/51484682](https://blog.csdn.net/AlbertFly/article/details/51484682 "https://blog.csdn.net/AlbertFly/article/details/51484682")