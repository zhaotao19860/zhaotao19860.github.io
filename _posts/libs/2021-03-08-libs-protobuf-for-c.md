---
layout: post
title: protobuf for c
date: 2021-03-08 11:16:00 +0800
category: Libs
---
[protobuf](https://github.com/protocolbuffers/protobuf)是google提供的一种数据序列化、反序列化工具，支持变长编码(数据压缩)、向后兼容(方便升级)、平台无关(方便移植)、语言无关(前后台解耦)等优秀特性。可用于数据存储、网络数据交互等场景。本文主要介绍protobuf在c语言编程中的安装步骤。<br/>
> protobuf安装包主要包括两部分：代码生成器(处理.proto) + 运行时库。<br/>
> c: 由于官方主代码库不提供c库，需要安装第三方库(插件形式)。<br/>
> c++/java/go/c#/python/js/object-c/ruby/php/dart: 均可使用官方库安装。

1. 安装protobuf(包括c++代码生成器 和 c++运行时库)
```bash
wget https://github.com/google/protobuf/releases/download/v3.6.0/protobuf-cpp-3.6.0.tar.gz
tar xvf protobuf-cpp-3.6.0.tar.gz
cd protobuf-3.6.0
./configure --prefix=/home/zhaotao/lib/   
make                                                                     
make install
```
2. 安装protobuf-c(包括c代码生成器 - 依赖protobuf-c++ 和c运行时库)
```bash
wget https://github.com/protobuf-c/protobuf-c/releases/download/v1.3.3/protobuf-c-1.3.3.tar.gz
tar xvf protobuf-c-1.3.3.tar.gz
cd protobuf-c-1.3.3
export PKG_CONFIG_PATH=/home/zhaotao/lib/protobuf/lib/pkgconfig  // 指定protobuf.pc文件所在
./configure
make
make install
```
3. 配置环境变量
```bash
####### add protobuf lib path ########
#(动态库搜索路径) 程序加载运行期间查找动态链接库时指定除了系统默认路径之外的其他路径
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/zhaotao/lib/
#(静态库搜索路径) 程序编译期间查找动态链接库时指定查找共享库的路径
export LIBRARY_PATH=$LIBRARY_PATH:/home/zhaotao/lib/
#执行程序搜索路径
export PATH=$PATH:/home/zhaotao/bin/
#c程序头文件搜索路径
export C_INCLUDE_PATH=$C_INCLUDE_PATH:/home/zhaotao/include/
#c++程序头文件搜索路径
export CPLUS_INCLUDE_PATH=$CPLUS_INCLUDE_PATH:/home/zhaotao/lib/include/
#pkg-config 路径
export PKG_CONFIG_PATH=/home/zhaotao/lib/pkgconfig/
```
4. 注意事项
<br/>—— PKG_CONFIG_PATH
```bash
如果没有使用export PKG_CONFIG_PATH=/home/zhaotao/lib/protobuf/lib/pkgconfig，在./configure这步可能会报错：
**No package 'protobuf' found**
这是因为Makefile中会用pkg-config命令检测环境变量，但是没有设置PKG_CONFIG_PATH，找不到protobuf.pc这个文件。
```
—— gcc编译选项
```bash
# 指定头文件
-I/home/zhaotao/lib/protobuf-c/include
# 指定库路径
-L/home/zhaotao/lib/protobuf-c/lib
# 指定库名称
-lprotobuf-c
```

