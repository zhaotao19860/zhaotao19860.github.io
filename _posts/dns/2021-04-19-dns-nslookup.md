---
layout: post
title: nslookup
date: 2021-04-19 11:44:00 +0800
category: DNS
---

#### 概述

**nslookup**(name server lookup)域名记录查询器，用来查询域名相关信息，包括linux版本及windows自带版本。

#### 安装(Linux)
```
yum install bind-utils
```
#### 安装(Windows)
```
自带
```
#### 语法
```
nslookup ?
nslookup [-option] [name | -] [server]
如果指定name，则进入非交互模式，直接返回结果；
如果不指定name，为空或者为-，则进入交互模式；
```
#### 查询指定域名a记录
```
nslookup domain [dns-server]
```
#### 查询其他记录类型
```
nslookup -type=mx domain [dns-server]
```
#### 反向解析
```
nslookup ip
```
#### 指定超时时间
```
nslookup -timeout=20 www.jd.com 
```
#### 打开调试信息
```
nslookup -debug www.jd.com 
```
