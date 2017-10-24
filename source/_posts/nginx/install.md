---
title: Linux下安装Nginx
tags: 
    - nginx
    - nginx安装
categories: 
    - Nginx
---
nginx在linux(本文基于centos7)下的安装虽说不复杂，但也偶尔会遇到零零散散的小问题，在此记录下nginx安装的简要步骤.

## 准备

### 下载nginx
在官网的[下载页面](http://nginx.org/en/download.html)，选择需要下载的版本, 推荐使用稳定版本.

### 上传到服务器

``` bash
$ scp nginx-1.12.2.tar.gz root@YouHost:~
```

### 解压

``` bash
$ tar zxvf nginx-1.12.2.tar.gz
```

## 安装

### OpenSSL
默认情况下，nginx是不支持ssl(也就是所谓的https),如果需要启用https的模块支持，则需要线安装openSSL,如果不需要，则跳过此步骤.

``` bash
$ yum install openssl
```

参考: [Nginx Https](http://nginx.org/en/docs/http/ngx_http_ssl_module.html)

### Nginx
进入nginx的根目录

``` bash
$ cd nginx-1.12.2
```

#### 配置

``` bash
$ ./configure --prefix=/usr/local/nginx --with-http_ssl_module
```

`--with-http_ssl_module` 参数说明配置启用ssl模块，按需要可启用不同的模块信息方法类似 `--with-模块名称`

`--prefix` 参数描述一个安装路径，nginx安装完成后，所有的配置、可运行的文件都在这个目录.

#### 编译

``` bash
$ make
```

#### 安装

``` bash
$ make install
```

#### 检查版本

``` bash
$ /usr/local/nginx/sbin/nginx -v
```

输出
```
nginx version: nginx/1.12.2
```

#### 检查安装模块

``` bash
$ /usr/local/nginx/sbin/nginx -V
```

输出
```
nginx version: nginx/1.12.2
built by gcc 4.4.7 20120313 (Red Hat 4.4.7-11) (GCC)
built with OpenSSL 1.0.1e-fips 11 Feb 2013
TLS SNI support enabled
configure arguments: --with-http_ssl_module
```
可以看到已经启用了ssl模块信息

### 启动

``` bash
$ /usr/local/nginx/sbin/nginx -c /usr/local/nginx/config/nginx.conf
```

### 注意事项

* 如果安装nginx之后, 发现还有额外模块没有安装，可以直接删除 `/usr/local/nginx` 目录，重新配置、编译、安装.
* 有时经常会遇到明明启用了SSL, 但是在配置文件中总是提示未启用改模块，这时候请确认，启动参数重的nginx是否是当前安装的，这里容易和系统默认的nginx弄混淆.
