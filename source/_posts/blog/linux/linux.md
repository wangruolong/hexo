---
title: Linux
author: 大大白
date: 2015-01-06 18:30:00
categories:
- 其他
- 服务器
tags: 
- Linux服务器基本配置
---

Linux服务器标配的基本环境。记录了服务器常用软件的安装方法，nginx、rar、https证书等。

<!-- more -->


## Nginx
### 安装
1. 需要先安装Nginx的依赖。`-y`是可选参数，意思是在安装的过程中遇到`Is this OK[y/d/N]`的时候，自动选择`y`。这个其实就是和Windows环境下的默认安装是同一个道理。
```
yum [-y] install gcc
yum [-y] install pcre-devel
yum [-y] install zlib zlib-devel
yum [-y] install openssl openssl-devel
```

2. 下载Nginx的安装包。
```
wget http://nginx.org/download/nginx-1.13.7.tar.gz
```

3. 解压安装包。
```
tar -xvf nginx-1.13.7.tar.gz
```

4. 运行configure。参数`--with-http_ssl_module`顾名思义，就是准备等下安装Nginx的时候带上ssl模块。
```
cd nginx-1.13.7/
./configure --with-http_ssl_module
```

5. 编译。
make

6. 安装。
make install

### 配置
location中的root和alias的最大区别是：root的结果是root+location，而alias的结果是alias替换location的路径。
需要特别注意的是alias后的的路径需要以`/`结尾

示例一
```
location /huan/ {
    alias /home/www/huan/;
}
```
在上面alias虚拟目录配置下，访问`http://www.xxx.com/huan/a.html`实际指定的是`/home/www/huan/a.html`。
注意：alias指定的目录后面必须要加上"/"，即/home/www/huan/不能改成/home/www/huan

上面的配置也可以改成root目录配置，如下，这样nginx就会去/home/www/huan下寻找`http://www.xxx.com/huan`的访问资源，两者配置后的访问效果是一样的！
```
location /huan/ {
    root /home/www/;
}
```

示例二
上面的例子中alias设置的目录名和location匹配访问的path目录名一致，这样可以直接改成root目录配置；那要是不一致呢？
再看一例：
```
location /web/ {
    alias /home/www/html/;
}
```

访问`http://www.colorgg.com/web`的时候就会去/home/www/html/下寻找访问资源。
这样的话，还不能直接改成root目录配置。
如果非要改成root目录配置，就只能在/home/www下将html->web（做软连接，即快捷方式），如下：
```
location /web/ {
    root /home/www/;
}
```

## rar解压工具
安装rar解压工具

## https证书
安装https证书
