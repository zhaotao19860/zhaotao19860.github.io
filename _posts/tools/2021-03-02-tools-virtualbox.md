---
title: Windows VirtualBox
date: 2021-03-02 09:55:00 +0800
category: Tools
---
[VirtualBox](https://www.virtualbox.org/)是一款开源的虚拟机软件，支持windows/linux/mac等多个系统。

### 软件下载
```bash
# virtualbox(开源)
https://www.virtualbox.org/wiki/Downloads
```
### 软件安装
```bash
双击
```
### 镜像下载
```bash
http://mirrors.aliyun.com/centos/7/isos/x86_64/
```
### 镜像安装
```bash
# 加载iso镜像
CentOS-7-x86_64-DVD-2009.iso
# 配置cpu、内存、硬盘、网络
4c/1g/20g/nat
# 开始安装
设置系统安装盘，设置root密码，最小化安装
```
### 登录虚拟机
```bash
consol方式：不需要配置；
ssh方式：如果网络连接方式选择nat，需要根据第8步配置端口映射；
```
### 系统配置
```bash
# 登录
console/ssh
# 查看网卡状态
ip a 
# 设置网卡随机启动
vi /etc/sysconfig/network-scripts/ifcfg-ens33 
ONBOOT=yes
# 关闭防火墙
systemctl stop firewalld
# 重启系统
reboot 
```
### 修改yum源
```bash
# 登录
console/ssh
# 安装wget
yum install -y wget
# 备份老的yum源
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
# 获取新的yum源
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
# 生成缓存
yum makecache
# 安装epel
yum install epel-release
```
### 端口映射
```bash
1.停止虚拟机；
2.设置端口映射；
3.启动虚拟机；
4.配置xshell--->ssh root@127.0.0.1:2222
```
<br/>![img](/assets/images/virtualbox/nat.png)
![img](/assets/images/virtualbox/port-map.png)