---
title: etcd
date: 2021-04-16 17:00:00 +0800
category: Tools
---
**etcd** 是一个高可用强一致性的键值对存储系统，用于共享配置和服务发现。

##### 单机
###### 安装
```
yum install etcd
```
###### 配置
```
vi /etc/etcd/etcd.conf
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"----无需修改
ETCD_LISTEN_CLIENT_URLS="http://localhost:2379"----localhost改为本机ip
ETCD_NAME="renminwang-etcd"----设置etcd名称
ETCD_ADVERTISE_CLIENT_URLS="http://localhost:2379"---localhost改
```
###### 服务
```
cat /usr/lib/systemd/system/etcd.service
```
###### 启动
```
systemctl start etcd.service
```
##### 集群
###### 服务
```
vi /usr/lib/systemd/system/etcd.service
[Unit]
Description=etcd server
After=syslog.target
[Service]
Type=simple
User=root
ExecStartPre=/bin/sh -c '/usr/bin/rm -rf /export/data/etcd/*'
ExecStart=/bin/sh -c '/usr/bin/etcd --name infra39 \ --initial-advertise-peer-urls http://10.226.193.39:2380 \ --listen-peer-urls http://10.226.193.39:2380 \
--listen-client-urls http://10.226.193.39:2379,http://127.0.0.1:2379 \ --advertise-client-urls http://10.226.193.39:2379 \
--initial-cluster-token etcd-cluster-test \
--initial-cluster \
infra40=http://10.226.193.40:2380,infra39=http://10.226.193.39:2380,infra69=http://10.226.193.69:2380 \
--initial-cluster-state existing \
--data-dir /export/data/etcd \
--quota-backend-bytes $((8*1024*1024*1024)) \
--auto-compaction-retention=1 \
--heartbeat-interval=200 \
--snapshot-count=50000 \
--max-snapshots=24 \
--max-wals=24 \
>>/export/log/etcd/etcd.log 2>&1'
#Restart=on-failure
RestartSec=5s
[Install]
WantedBy=multi-user.target
```
###### 清理
```
/usr/bin/rm -rf /export/data/etcd/
```
###### 创建log文件
```
mkdir /export/log/etcd/
```
###### 启动
```
systemctl start etcd
```
##### etcdctl
###### 环境变量
```
export ETCDCTL_API=3
```
###### 新增
```
etcdctl  --endpoints=10.226.133.67:2379 put /zhaotao/1 1
注意：如果插入时key不以/开头，查询时开头也不能带/
```
###### 查询
```
etcdctl  --endpoints=10.226.133.67:2379 get dns-dev-bamboo --prefix 查询指定前缀
etcdctl  --endpoints=10.226.133.67:2379 get --prefix “"
注意：如果插入时key不以/开头，查询时开头也不能带/
```
###### 问题
```
解决mvcc: database space exceeded异常
#使用API3
export ETCDCTL_API=3
# 查看告警信息，告警信息一般 memberID:8630161756594109333 alarm:NOSPACE
etcdctl --endpoints=http://10.226.133.67:2379 alarm list
# 获取当前版本
rev=$(etcdctl --endpoints=http://10.226.133.67:2379 endpoint status --write-out="json" | egrep -o '"revision":[0-9]*' | egrep -o '[0-9].*')
# 压缩掉所有旧版本
etcdctl --endpoints=http://10.226.133.67:2379 compact $rev
# 整理多余的空间
etcdctl --endpoints=http://10.226.133.67:2379 defrag
# 取消告警信息
etcdctl --endpoints=http://10.226.133.67:2379 alarm disarm
# 自动压缩
启动时增加--auto-compaction-retention=1(只保存1小时的历史记录)
```