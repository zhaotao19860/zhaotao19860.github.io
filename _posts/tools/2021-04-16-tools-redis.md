---
layout: post
title: redis
date: 2021-04-16 17:00:00 +0800
category: Tools
---

#### 概述
**redis** (REmote DIctionary Server),是用c语言开发的，分布式，key-value内存数据库，并支持持久化。
#### 安装
```bash
yum install redis
```
#### 启动
```bash
redis-server /etc/redis.conf
```
#### 连接
```bash
redis-cli -h host -p port -a password
```
#### 切换库
```bash
select 1
```
#### 新增
```bash
set default '{ "qps_limit": 10000, "bps_limit": 2000000000, "max_domain_len": 0, "proxy": 0}'
```
#### 删除
```bash
del default
```
#### 查询
```bash
get default
```