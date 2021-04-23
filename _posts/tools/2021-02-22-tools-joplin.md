---
title: Joplin
date: 2021-02-22 11:40:00 +0800
category: Tools
---

[Joplin](https://github.com/laurent22/joplin)是一款开源的笔记类软件，支持Windows, macOS, Linux, Android and iOS等多种客户端。本文提供一套自有且完全开源的笔记系统的搭建过程，基于[wsgidav](https://github.com/mar10/wsgidav)作为服务端，提供同步功能；joplin作为客户端，提供编辑及浏览功能。

# 服务端
## 安装
```bash
yum -y install epel-release        -- 增加yum源
yum install -y python36            -- 安装python3
yum install -y python36-setuptools -- 安装easy_install
easy_install-3.6 pip               -- 安装pip3
pip3 install cheroot wsgidav       -- 安装cheroot/wsgidav
pip3 install python-pam            -- 安装python-pam
```
## 配置
```bash
useradd -m tom -- 新增用户
passwd tom     -- 设置密码
mkdir -p /root/wsgidav
```
## 启动
```bash
su tom
wsgidav --host=0.0.0.0 --port=8002 --root=/root/wsgidav --auth=pam-login
```
## 服务化
### service

| vi /etc/systemd/system/wsgidav.service                       |
| :----------------------------------------------------------- |
| [Unit]<br/>Description=WsgiDAV WebDAV server<br/>After=network.target<br/><br/>[Service]<br/>Type=simple<br/>ExecStart=/bin/sh -c 'wsgidav --host=0.0.0.0 --port=8002 --root=/root/wsgidav --auth=pam-login'<br/>StandardOutput=syslog<br/>StandardError=syslog<br/>SyslogIdentifier=wsgidav_service<br/><br/>[Install]<br/>WantedBy=muti-user.target |

### rsyslog

| vi /etc/rsyslog.d/wsgidav_service.conf                       |
| :------------------------------------------------------------ |
| if $programname == 'wsgidav_service' then /var/log/wsgidav.log<br/>& stop |

### 启动
```bash
touch /var/log/wsgidav.log
chown root:adm /var/log/wsgidav.log
systemctl daemon-reload
systemctl restart rsyslog
su tom
systemctl start wsgidav
systemctl status wsgidav
tail -f /var/log/wsgidav.log
```

# 客户端
## windows
```bash
Download:
https://github.com/laurent22/joplin/releases ---> Joplin-Setup-xxx.exe
Config:
joplin ---> Tools ---> Options ---> Synchronisation
Synchronisation target: WebDAV
WebDAV URL: http://x.x.x.x:8002
WebDAV username: tom
WebDAV password: xxx
```

## iphone
```bash
Download:
AppStore ---> Joplin
Config:
joplin ---> Configuration ---> Synchronisation
Synchronisation target: WebDAV
WebDAV URL: http://x.x.x.x:8002
WebDAV username: tom
WebDAV password: xxx
```
