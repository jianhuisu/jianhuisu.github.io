---
title : ffmpeg安装过程中涉及到的知识点
categories : 
 - Linux 
tags :
	- Linux
---

昨天协助同事安装 ffmpeg 编码mp3文件 遇到几个问题,记录一下

1 configure时无法通过,依赖问题主要依赖 lame、x264。

    这个没什么好说的,依赖什么装什么,可以使用yum安装就用yum ,无法使用yum就 download 、configure、make && make install 素质三连
    有些工具安装完事后需要自己手动创建软链,注意安装完依赖后which一下
    注意下载时的三个点,这几点的相关信息一般在configure的错误提示或者日志里都有记录,出现问题优先从这里寻找帮助,然后是 baidu google
        
        安装目录的指定  
        尽量去官方网站下载源码包
        注意源码包的版本要相互匹配
        
        ffmpeg 4.1
        lame 3.100
        x264 最新版    
    
    ./configure --prefix=/usr/local/ffmpeg --enable-libmp3lame --enable-libx264 --enable-gpl --disable-x86asm
    
                
2 安装完依赖,安装 ffmpeg,这个过程没碰见什么大问题,问题出现在运行ffmpeg时

    [root@iz2zeiaj6kd5kgjklk2er2z ~]# ffmpeg -i 075b352cd6b628c1b964ec661127d4fa.mp3 self.mp3
    ffmpeg: error while loading shared libraries: libmp3lame.so.0: cannot open shared object file: No such file or directory
                 
别慌,这个是路径问题,翻译过来就是不能在共享库中找到 libmp3lame.so.0 文件.共享库也叫动态链接库
(什么是动态链接库与静态链接库:https://blog.csdn.net/li1914309758/article/details/82145023#comments)
这个问题是怎么产生的呢?

这里需要了解加一个文件 `/etc/ld.so.conf`

/etc/ld.so.conf 此文件记录了编译时使用的动态库的路径，也就是加载so库的路径。默认情况下，编译器只会使用/lib和/usr/lib这两个目录下的库文件，而通常通过源码包进行安装时，如果不
指定--prefix会将库安装在/usr/local目录下，而又没有在文件/etc/ld.so.conf中添加/usr/local/lib这个目录>。这样虽然安装了源码包，但是使用时仍然找不到相关的.so库，就会报错。也就是说系统不知道安装了源码包。
对于此种情况有2种解决办法：

    （1）在用源码安装时，用--prefix指定安装路径为/usr/lib。这样的话也就不用配置PKG_CONFIG_PATH
     (2) 直接将路径/usr/local/lib路径加入到文件/etc/ld.so.conf文件的中。在文件/etc/ld.so.conf中末尾直接添加：/usr/local/lib（这个方法给力！编辑完后注意使用 ldconfig 刷新缓存）
     (3) 如果想重新编译 可以创建软链
    
ldconfig

ldconfig 这个程序，位于/sbin下，它的作用是将文件/etc/ld.so.conf列出的路径下的库文件缓存到/etc/ld.so.cache以供使用，因此当安装完一些库文件，或者修改/etc/ld.so.conf增加了库的新的搜索路径，需要运>行一下ldconfig，使所有的库文件都被缓存到文件/etc/ld.so.cache中，如果没做，可能会找不到刚安装的库。
       
所以,我们看一下`/etc/ld.so.conf`文件

    root@iz2zeiaj6kd5kgjklk2er2z ~]# cat /etc/ld.so.conf
    include ld.so.conf.d/*.conf
    /lib
    /usr/lib
    /usr/lib64
    /usr/local/lib64

我们使用`ldd`查看一下 ffmpeg 动态链接库的依赖关系

    [root@iz2zeiaj6kd5kgjklk2er2z ~]# ldd `which ffmpeg`
    	linux-vdso.so.1 =>  (0x00007ffc3b9a0000)
    	libm.so.6 => /usr/lib64/libm.so.6 (0x00007f34d9d94000)
    	libxcb.so.1 => /usr/lib64/libxcb.so.1 (0x00007f34d9b6b000)
    	libxcb-shm.so.0 => /usr/lib64/libxcb-shm.so.0 (0x00007f34d9967000)
    	libxcb-shape.so.0 => /usr/lib64/libxcb-shape.so.0 (0x00007f34d9763000)
    	libxcb-xfixes.so.0 => /usr/lib64/libxcb-xfixes.so.0 (0x00007f34d955a000)
    	libbz2.so.1 => /usr/lib64/libbz2.so.1 (0x00007f34d934a000)
    	libz.so.1 => /usr/lib64/libz.so.1 (0x00007f34d9134000)
    	libiconv.so.2 => /usr/local/lib/libiconv.so.2 (0x00007f34d8e4d000)
    	liblzma.so.5 => /usr/lib64/liblzma.so.5 (0x00007f34d8c27000)
    	libmp3lame.so.0 => /lib64/libmp3lame.so.0 (0x00007f34d8999000)
    	、、、
    	libx264.so.157 => not found
    	、、、
    	libpthread.so.0 => /usr/lib64/libpthread.so.0 (0x00007f34d877c000)
    	libc.so.6 => /usr/lib64/libc.so.6 (0x00007f34d83b9000)
    	/lib64/ld-linux-x86-64.so.2 (0x0000563a704e3000)
    	libXau.so.6 => /usr/lib64/libXau.so.6 (0x00007f34d81b4000)

看到没 ffmpeg在链接动态库时去`usr/lib64/`目录下寻找,但是 `libmp3lame.so.0` 被安装在了 `/usr/local/lib/`
创建一个软链
        
        ln -s /usr/local/lib/libmp3lame.so.0.0.0 /usr/lib64/libmp3lame.so.0
        

然后运行 ffmpeg
ffmpeg: error while loading shared libraries: libx264.so.157: cannot open shared object file: No such file or directory
再创建

        ln -s /usr/local/lib/libx264. /usr/lib64/libx264.so.157
                
再运行

    [root@iz2zeiaj6kd5kgjklk2er2z ~]# ffmpeg
    ffmpeg version 4.1 Copyright (c) 2000-2018 the FFmpeg developers
      built with gcc 4.8.5 (GCC) 20150623 (Red Hat 4.8.5-36)
      configuration: --prefix=/usr/local/ffmpeg --enable-libmp3lame --enable-libx264 --enable-gpl --disable-x86asm
      libavutil      56. 22.100 / 56. 22.100
      libavcodec     58. 35.100 / 58. 35.100
      libavformat    58. 20.100 / 58. 20.100
      libavdevice    58.  5.100 / 58.  5.100
      libavfilter     7. 40.101 /  7. 40.101
      libswscale      5.  3.100 /  5.  3.100
      libswresample   3.  3.100 /  3.  3.100
      libpostproc    55.  3.100 / 55.  3.100
    Hyper fast Audio and Video encoder
    usage: ffmpeg [options] [[infile options] -i infile]... {[outfile options] outfile}...            

收工  
    
参考资料

    ldd 命令简介  https://www.cnblogs.com/zhangjxblog/p/7776556.html
    解决运行 ffmpeg 时动态库 not found 问题 https://www.cnblogs.com/joshua317/articles/5478622.html
    ffmpeg 依赖的安装 http://blog.chinaunix.net/uid-11344913-id-3930867.html
    ffmpeg 官方下载地址 http://www.ffmpeg.org/releases/
   ffmpeg 依赖安装x264 https://www.cnblogs.com/cxchanpin/p/6943221.html 
