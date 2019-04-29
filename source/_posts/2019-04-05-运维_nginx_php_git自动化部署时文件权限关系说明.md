---
title : nginx_php_git自动化部署时文件权限关系说明
categories : 
 - 运维 
tags :
    - 服务器运维与架构
---

## git部署的权限一致性

执行git拉取命令的用户与项目文件所有者保持一致
php-fpm woker进程的owner与项目文件的owner保持一致
nginx worker进程的owner对项目文件具有读权限，最好也与owner身份保持一致(maybe not safe)
当然如果使用ftp上传文件，那么文件的属性设置必须与项目的owner保持一致


Tips:使用git拉取时，使用哪个用户执行拉取，那么文件的所有者就为命令的执行者，例如

创建仓库时 使用 machine_su 拉取仓库 ，此时仓库所有文件属于 machine_su,但是，后期更新时有一次使用root进行拉取，此时新增/修改的文件则属于root。
可以这样理解 git在拉取时实际为先删除原有文件，然后新建文件，那么新建文件的所有者自然是git命令的执行者 . 而未修改的文件则权限无变化


nginx worker进程的执行者指定       nginx.conf
php-fpm worker进程的执行者指定     php-fpm.conf/或者php-fpm.d/www.conf
Tips：只有当master进程的执行者为root时，才可以指定worker进程执行者的身份，否则worker/master的owner相同
