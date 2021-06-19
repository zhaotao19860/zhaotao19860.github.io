---
title: git版本升级
date: 2021-06-19 12:28:00 +0800
category: Tools
---

linux系统本身自带的git版本都是非常低的，比如centos7-git-1.8.3，低版本的git会存在功能及性能的问题，所以最好升级。

#### 第一步
下载最新的发布版本，[github官网](https://github.com/git/git/git/releases]
```bash
cd /usr/src/
wget https://github.com/git/git/archive/v2.32.0.tar.gz
tar -xf v2.32.0.tar.gz
```
#### 第二步
安装依赖
```bash
yum install curl-devel expat-devel gettext-devel openssl-devel zlib-devel gcc perl-ExtUtils-MakeMaker
```
#### 第三步
删除旧版本
```bash
yum remove git
```
#### 第四步
解决编译过程中可能出现的openssl编译出错的问题，找到openssl位置
```bash
which openssl
如果仍旧不对，需要尝试/usr/local/openssl
```
#### 第五步
配置及编译
```bash
cd git-2.32.0
autoconf
./configure --prefix=/usr/local/git --with-openssl=/usr/bin/openssl
make all
make install
```
#### 第六步
添加到系统path路径
```bash
ln -s /usr/local/git/bin/git /usr/bin/git
```
#### 第七步
升级man手册
```bash
cd /usr/src/
wget https://mirrors.edge.kernel.org/pub/software/scm/git/git-manpages-2.32.0.tar.gz
tar -xf git-manpages-2.32.0.tar.gz
cd git-manpages-2.32.0
cp man1/* /usr/local/share/man/man1/
cp man5/* /usr/local/share/man/man5/
cp man7/* /usr/local/share/man/man7/
```