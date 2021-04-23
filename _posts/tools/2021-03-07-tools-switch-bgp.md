---
title: 交换机bgp配置
date: 2021-03-07 11:47:00 +0800
category: Tools
---
BGP(Border Gateway Protocol)，即边际网关协议，主要用于AS(autonomous system)之间的路由信息交换(EBGP)，也可用于大型的AS内部(IBGP)。提供BGP服务的实体叫做BGP router，而与BGP router连接的对端叫做BGP peer。推荐阅读[BGP漫谈](https://zhuanlan.zhihu.com/p/25433049)。本文主要讲解基于华为三层交换机[H3C S5830V2]的配置方法。

### 登录交换机
```bash
ssh tptest@1.1.1.1  密码：xxx
```
### 进入系统视图
```bash
system-view
```
### 创建bgp并设置ipv6邻居
```bash
bgp 65535
peer 2001::104 as-number 65535
address-family ipv6
peer 2001::104 enable
balance 2
display bgp peer ipv6 unicast
display bgp routing-table ipv6
quit
quit
```
### 创建bgp并设置ipv4邻居
```bash
bgp 65535
peer 2.2.3.104 as-number 65535
address-family ipv4
peer 2.2.3.104 enable
balance 2
display bgp peer ipv4 unicast
display bgp routing-table ipv4
quit
quit
```
### 创建并设置vlan
```bash
vlan 11
quit
interface Vlan-interface 11
ipv6 address 2001::1/64
ip address 2.2.3.1/24
ping ipv6 2000::100
ping ip 2.2.1.10
quit
```
### 将端口加入vlan
```bash
interface Ten-GigabitEthernet 1/0/5
port link-type access
port access vlan 11
quit
```