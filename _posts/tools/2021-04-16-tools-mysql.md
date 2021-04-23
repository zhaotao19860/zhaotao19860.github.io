---
title: mysql
date: 2021-04-16 17:00:00 +0800
category: Tools
---

**mysql**用于操作mysql数据库的命令行工具。

#### 连接
```bash
mysql -h10.226.133.67 -P3306 -uroot -pxwS7UDKwHxff  db_dns_master
-h:主机
-P:端口
-u:用户名
-p:密码
db_dns_master:数据库名称
```
#### 查看数据库
```bash
show databases;
```
#### 选择数据库
```bash
use db_dns_master;
```
#### 查看所有表
```bash
show tables;
```
#### 显示表结构
```bash
describe dns_domain;
```
#### 建库
```bash
create database 库名;
```
#### 建表
```bash
use 库名;
create table 表名(字段设定列表);
```
#### 清空表
```bash
delete from 表名;
truncate table 表名;
```
#### 查询表
```bash
select * from 表名;
```
#### 删除表
```bash
use 库名;
drop table 表名;
```
#### 删除库
```bash
drop database 库名;
```
