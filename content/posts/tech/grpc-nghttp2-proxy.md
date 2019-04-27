---
title: "grpc（2）：Centos 安装 nghttp2 做 grpc 的http2 代理"
date: 2019-04-21T22:02:57+08:00
hidden: true
draft: true
tags: [golang, go, goim]
keywords: [tsingson]
description: "goim 架构与定制"
slug: "go-skil-01"
---




# grpc（2）：Centos 安装 nghttp2 做 grpc 的http2 代理

## 1，nghttp2
和nginx 名字比较像，但是是一个c的llib库。本身也可做http服务。 
也可以做代理服务器，支持ssl。 
之前也做过测试了 
http://blog.csdn.net/freewebsys/article/details/58584294 
因为nginx 是不支持 upstream 的http2 转发请求的。 
而且nginx 也没有计划开发这个。 
而haproxy 是支持 tcp 做代理的。对http2 的协议也是不支持的。 
以后还打算做一个 grpc的网关。 
必须要能支持http2的协议。而且还能够代理grpc。 
找了半天就找到了一个nghttp2.。

## 2，下载安装
官方网站： 
https://nghttp2.org/ 
https://github.com/nghttp2/nghttp2 
官方文档是在ubuntu或者debian上面进行安装的。 
实际上也可以在centos上面进行安装。 
参考： 
https://kirk91.github.io/2017/02/22/build-nghttp2-on-centos/ 
直接下一步下一步就可以了。 
安装依赖库：
```
sudo yum -y groupinstall "Development Tools"
sudo yum -y install openssl-devel libxml2-devel libev-devel jemalloc-devel python-devel
wget https://c-ares.haxx.se/download/c-ares-1.12.0.tar.gz -O /tmp/c-ares.tar.gz
mkdir -p /tmp/c-ares
tar -zxvf /tmp/c-ares.tar.gz -C /tmp/c-ares --strip-components=1
cd /tmp/c-ares && ./configure --libdir=/usr/lib64
make
sudo make install
wget http://www.digip.org/jansson/releases/jansson-2.9.tar.gz -O /tmp/jansson.tar.gz
mkdir -p /tmp/jansson
tar -zxvf /tmp/jansson.tar.gz -C /tmp/jansson --strip-components=1
cd /tmp/jansson && ./configure --libdir=/usr/lib64
make
make check
sudo make install
```
安装nghttp2服务。
```
wget https://github.com/nghttp2/nghttp2/releases/download/v1.19.0/nghttp2-1.19.0.tar.gz -O /tmp/nghttp2.tar.gz
mkdir -p /tmp/nghttp2
tar -zxvf /tmp/nghttp2.tar.gz -C /tmp/nghttp2 --strip-components=1
cd /tmp/nghttp2 && ./configure --enable-app
make
sudo make install
```
没有错误就是编译成功了。

## 3，启动服务
网上的文档比较少 
https://nghttp2.org/documentation/package_README.html 
配置就直接按照proxy进行配置即可。 
https://nghttp2.org/documentation/nghttpx-howto.html 
但是发现几个比较坑的地方。使用命令行的参数和配置文件的不太一样。 
结果是配置文件的可以使用。参数定义的比较怪异。 
我花了一个下午的时间折腾，重要明白了咋配置了。 
nghttpx.conf
```
frontend=0.0.0.0,5000;no-tls
backend=127.0.0.1,50051;/;proto=h2
backend=127.0.0.1,50051;/helloworld.Greeter/;proto=h2
backend=127.0.0.1,50052;/aaa/;proto=h2
#http2-proxy=no
workers=10
accesslog-file=/data/nghttp/log/access.log
errorlog-file=/data/nghttp/log/errorlog.log
```
首先是frontend 配置，不使用tls进行访问的话一定要加上。 
否则就需要增加key 和 crt 文件，而且访问的时候要使用https。 
backend端，一定不要加上tls，否则会报502 错误。 
而且对于backend的grpc服务来说一定要加上 proto=h2 参数。 
强制协议是http2的。否则也报502 错误。

4，启动服务
nghttpx --conf nghttpx.conf 
1
从访问日志里面看：
```
10.0.2.2 - - [01/Mar/2017:04:29:18 -0500] "POST /helloworld.Greeter/SayHello HTTP/2" 200 32 "-" "token=xxxxx grpc-java-netty/1.1.2"
10.0.2.2 - - [01/Mar/2017:04:29:18 -0500] "POST /helloworld.Greeter/SayHello HTTP/2" 200 32 "-" "token=xxxxx grpc-java-netty/1.1.2"
10.0.2.2 - - [01/Mar/2017:04:29:18 -0500] "POST /helloworld.Greeter/SayHello HTTP/2" 200 32 "-" "token=xxxxx grpc-java-netty/1.1.2"
10.0.2.2 - - [01/Mar/2017:04:29:18 -0500] "POST /helloworld.Greeter/SayHello HTTP/2" 200 32 "-" "token=xxxxx grpc-java-netty/1.1.2"
10.0.2.2 - - [01/Mar/2017:04:29:18 -0500] "POST /helloworld.Greeter/SayHello HTTP/2" 200 32 "-" "token=xxxxx grpc-java-netty/1.1.2"
10.0.2.2 - - [01/Mar/2017:04:29:18 -0500] "POST /helloworld.Greeter/SayHello HTTP/2" 200 32 "-" "token=xxxxx grpc-java-netty/1.1.2"
```
可以看到请求。 
其中/helloworld.Greeter/SayHello 代表grpc的包名，接口名和方法名。

5，总结
本文的原文连接是: http://blog.csdn.net/freewebsys/article/details/59112145 未经博主允许不得转载。 
博主地址是：http://blog.csdn.net/freewebsys

nghttp2 是不错的grpc代理服务器，可以做简单的负载均衡。 
同时保持http2的链接。 
是一个不错的grpc gateway 解决方案。唯一不足的地方是用 c++ 编写的。 
要是再修改成一个可以做权限，数据统计的gateway 修改起来还是有点难度的。本人不是搞c++ 开发的。
--------------------- 
作者：freewebsys 
来源：CSDN 
原文：https://blog.csdn.net/freewebsys/article/details/59112145 
版权声明：本文为博主原创文章，转载请附上博文链接！