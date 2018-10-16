---
title: github hook 自动化部署hexo博客
tags:
	- 运营
---

	<?php
	
	error_reporting(E_ERROR | E_PARSE);
	date_default_timezone_set('PRC');
	
	$webRoot = '/home/www/';
	$token = 'Your token';
	$user = 'www';
	$group = 'www';
	
	$payload = file_get_contents('php://input');
	$json = json_decode($payload, true);
	$repo = $json['repository']['name'];
	
	$signature = 'sha1='.hash_hmac("sha1", $payload, $token);
	if(strcmp($_SERVER['HTTP_X_HUB_SIGNATURE'],$signature) !== 0){
	    echo "X-Hub-Signature is not match!";
	    exit(0);
	}
	
	if ($json['ref'] === 'refs/heads/master') {
	
	    $projectDirectory = $webRoot.$repo.'/';
	
	    //$output = shell_exec("/var/www/qadoor/qadoor_site/public/hook/deploy.sh");
	    $output = exec("cd $projectDirectory && git pull origin master && hexo g");
	
	    //log the request
	    setLog($output."success !");
	    echo "success";
	
	}
	
	function setLog($msg)
	{
	    $fd = fopen(__DIR__."/hook.log",'a+');
	    fwrite($fd,date("Y-m-d H:i",time())."    ".$msg."\n");
	    fclose($fd);
	}
