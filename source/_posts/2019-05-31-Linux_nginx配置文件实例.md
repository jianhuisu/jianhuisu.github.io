---
title : Nginx配置文件实例
categories : 
 - Linux 
tags :
	- Nginx
---

直接看注释

    server {
    
            listen       80; 
            server_name  ~^host\.(?<project>[^\.]+?)\.com$;
        
        # 网站的根目录 未做特殊处理的情况下 用户可以根据相对路径访问网站根目录下的任意文件 跨过你所设置的核心入口 index.php 所以一般项目会将web根目录隐藏在核心文件的下层 
            root /Users/www/$project;
            index index.html; 
    
        # 隐藏响应头中的有关操作系统和Nginx服务器版本号的信息，保障安全性
        server_tokens off;
    
        location = / {
             proxy_pass http://127.0.0.1:80;
            }    
    
            access_log  logs/host.$project.access.log;
            error_page   500 502 503 504  /50x.html;
    
            location = /50x.html {
                root   html;
            }   
    
        location ~* (jpg|jpeg|png|gif|txt|pdf|css|js|ico)$
        {
            
            expires 30d;
    
            # 取消对静态文件的日志记录
            access_log off;	
        }
    
        # 禁止访问 根目录下app目录中的所有内容 向客户端返回 403 forbidden
        location ~* /app
        {
            allow 127.0.0.1;
            deny all;
        }
        
    
        location ~* /break/
        {
            # rewrite 重写之后,终止本模块中向下执行(即下面的return 402 不会执行) 直接去进行资源定位 不会再次进行 location 匹配 
            rewrite ^/break/(.*) /test/$1 break;
            return 402; 
        }
        
        location ~* /last/
        {
           # rewrite 之后 停止本模块语句的向下执行 重新进行location 匹配
           rewrite ^/last/(.*) /test/$1 last;
           return 502;
        }
    
        location ~* /test/
        {
           return 508;
        }
    
        # http://host.sll.com/nginx_status  可以通过这种访问方式查看 nginx 状态
        location /nginx_status{
            stub_status on;
            access_log off;
            allow 127.0.0.1;
            deny all;
        }
        
    
    
    }

location 中可用的表达式

    -f和!-f用来判断是否存在文件
    -d和!-d用来判断是否存在目录
    -e和!-e用来判断是否存在文件或目录
    -x和!-x用来判断文件是否可执行
    
location 匹配规则

语法规则： location [=|~|~*|^~] /uri/ { … }

    = 开头表示精确匹配
    ^~ 开头表示uri以某个常规字符串开头，理解为匹配 url路径即可。nginx不对url做编码，因此请求为/static/20%/aa，可以被规则^~ /static/ /aa匹配到（注意是空格）。
    ~ 开头表示区分大小写的正则匹配
    ~*  开头表示不区分大小写的正则匹配
    !~和!~*分别为区分大小写不匹配及不区分大小写不匹配 的正则
    / 通用匹配，任何请求都会匹配到。
    多个location配置优先级：按照精准度由高到低 = > ~ > / , 同级的按照在文件中的排列顺序, 匹配成功后停止匹配
 
如何查看nginx已经安装了哪些模块 

    nginx -V  

如果有一个模块没有安装 那么可以编译追加,注意 不要使用 make install 这样是覆盖安装	
    
    