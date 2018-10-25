---
layout:     post
title:      PHP编程经验
keywords: "session 自定义session"
catalog: true
tags:
    - PHP
---
# 如何自定义session存储过程 #

想要自定义session的存储过程，必须理解session的默认存储机制，[官方手册](http://php.net/manual/zh/function.session-set-save-handler.php)的介绍最具有权威性。


## 默认情况下Session的处理方式 ##

1. 客户端(通常为浏览器)向服务器发起请求
1. PHP检查http请求头中是否存在session_id,不存在则重新生成 
1. 调用 `session_start()` 函数，php会在内部顺序调用 `SessionHandler` 的 `open()` 、`read()`方法,调用 `read()`方法时，PHP 会将从session文件中读取的内容自动反序列化返回为字符串，并填充为 $_SESSION 超级全局变量的值。
1. 操作$_SESSION变量中的数据
1. 脚本结束后，php会自动顺序调用 `SessionHandler` 的 `write()` 、`close()` 方法，调用`write()`,方法时，PHP会把 $_SESSION中 数据序列化为字符串写入对应的session文件
1. 把session_id存储在cookie中返回至浏览器
1. 在脚本执行过程中，PHP会自动按照一定几率调用  `SessionHandler` 的 `gc()`函数，回收过期session文件

## 为什么要自定义Session处理方式 ##

将一套代码部署到多台服务器时，使用默认的会话存储方式会产生A服务器需要读取B服务器上的session文件，显然这是不合理的，所以将所有的session信息存储单台服务器是一个更好的选择。而且，将session信息存储到mysql或者redis，比使用文件来存储信息性能更高


## 使用session\_set\_save_handler() ##

session\_set\_save_handler() 函数允许定义 user-level 的会话管理函数，也就是说，成功注册自定义hanlder后，PHP会调用我们注册的handler。函数介绍

	/**
	 * (PHP 5.4)<br/>
	 * Sets user-level session storage functions
	 * @link http://php.net/manual/en/function.session-set-save-handler.php
	 * </p>
	 * @param SessionHandlerInterface $session_handler An instance of a class implementing SessionHandlerInterface, such as SessionHandler,
	 * to register as the session handler. Since PHP 5.4 only.
	 * @param bool $register_shutdown [optional] Register session_write_close() as a register_shutdown_function() function.
	 * @return bool true on success or false on failure.
	 */
	function session_set_save_handler (SessionHandlerInterface $session_handler, $register_shutdown = true) {}


开始上代码

	define('WEB_ROOT',__DIR__);

	class DbSession extends \SessionHandler
	{
	    private $savePath;
	
	    function __construct()
	    {
	        echo __FUNCTION__."<br/>";
	        register_shutdown_function('session_write_close');
	    }
	
	    function open($savePath, $sessionName)
	    {
	        echo __FUNCTION__."<br/>";
	        $this->savePath = WEB_ROOT.'/runtime/session';
	        if (!is_dir($this->savePath)) {
	            mkdir($this->savePath, 0777);
	        }
	
	        return true;
	    }
	
	    function close()
	    {
	        echo __FUNCTION__."<br/>";
	        return true;
	    }
	
	    function read($id)
	    {
	        echo __FUNCTION__."<br/>";
	
	        $file = "$this->savePath/sess_$id";
	        if(file_exists($file)){
	            return (string)@file_get_contents($file);
	        }
	
	        return '';
	    }
	
	    function write($id, $data)
	    {
	        echo __FUNCTION__."<br/>";
	        return file_put_contents("$this->savePath/sess_$id", $data) === false ? false : true;
	    }
	
	    function destroy($id)
	    {
	        echo __FUNCTION__."<br/>";
	        $file = "$this->savePath/sess_$id";
	        if (file_exists($file)) {
	            unlink($file);
	        }
	
	        return true;
	    }
	
	    function gc($maxlifetime)
	    {
	        echo __FUNCTION__."<br/>";
	        foreach (glob("$this->savePath/sess_*") as $file) {
	            if (filemtime($file) + $maxlifetime < time() && file_exists($file)) {
	                unlink($file);
	            }
	        }
	
	        return true;
	    }
	
	}
	
	$obj = new DbSession();
	session_set_save_handler($obj);
	@session_start();

	var_dump($_SESSION);
	$_SESSION['name'] = 'machine_su';
	
	echo "script has finished</br>";
	exit;

通过观察输出的顺序，也可以印证文章开始描述的会话存储过程。把上述代码中的文件操作改为 mysql操作，就是session共享的原理。
附上session\_table
	
	CREATE TABLE session
	(
	    id CHAR(40) NOT NULL PRIMARY KEY,
	    expire INTEGER,
	    data BLOB
	)

## 易错点 ##

### session.auto_start ###

使用了自定义的session机制，php.ini 中的session.auto_start，必须设置为false。否则自定义的session机制失效，仍旧采用默认处理机制

### CLI模式不能使用session ###

cookie和session都是需要通过http协议请求头来创建的，也就是说需要浏览器发起创建，属于CGI模式，CLI模式区别于CGI模式。

### session过期时间 ###

如果不同服务器上的脚本具有不同的 session.gc_maxlifetime 数值但是共享了同一个地方存储会话数据，则具有最小数值的脚本会清理数据，木桶原理

### register\_shutdown\_function ###

`register_shutdown_function()` 注册函数为队列形式，按照注册顺序调用


> When using objects as session save handlers, it is important to register the shutdown function with PHP to avoid unexpected side-effects from the way PHP internally destroys objects on shutdown and may prevent the write and close from being called. Typically you should register 'session\_write\_close' using the register\_shutdown\_function() function.


思考：

1. 如果将session存储介质变更为mysql，是否可以将 session.gc_probability 和 session.gc_divisor 调满
1. 如何实时统计在线人数



