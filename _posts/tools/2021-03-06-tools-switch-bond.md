---
layout: post
title: 交换机端口聚合配置
date: 2021-03-06 11:13:00 +0800
category: Tools
---
端口聚合，又称为链路聚合，就是将物理上的多个端口配置到一起，形成逻辑上的一个端口，使之具有：增大带宽、负载均衡、冗余链路的优点。下面以华为三层交换机[H3C S5830V2]为例，说明端口聚合的配置过程。
1. 登录交换机
```bash
ssh tptest@1.1.1.1  密码：xxxx
```
2. 进入系统视图
```bash
system-view
```
3. 创建聚合口104
```bash
interface Bridge-Aggregation 104 #创建并进入二层聚合接口视图
port link-type access #配置为接入模式
port access vlan 11 #放行vlan
mode lacp #模式
dis this #查看配置
quit #退出二层聚合视图
```
>端口类型：<br/>
>**access**: 接入链路，指的交换机到用户设备的链路，即是接入到户，可以理解为由交换机向用户的链路，且只能属于1个VLAN。例如：当一个端口属于vlan 10时，那么带着vlan 10的数据帧会被发送到交换机这个端口上，当这个数据帧通过这个端口时，vlan 10 tag 将会被剥掉，到达用户电脑时，就是一个以太网的帧。而当用户电脑发送一个以太网的帧时，通过这个端口向上走，那么这个端口就会给这个帧加上一个vlan 10 tag。而其他vlan tag的帧则不能从这个端口上下发到电脑上。<br/>
>**trunk**: 干道链路，指的交换机到上层设备如路由器的链路，可以理解为向广域网走的链路，可以允许多个VLAN通过,可以接收和发送多个VLAN 报文。一个trunk端口可以拥有一个主vlan和多个副vlan，这个概念可以举个例子来理解：例如：当一个trunk端口有主vlan 10 和多个副vlan11、12、30时，带有vlan 30的数据帧可以通过这个端口，通过时vlan 30不被剥掉；当带有vlan 10的数据帧通过这个端口时也可以通过。如果一个不带vlan 的数据帧通过，那么将会被这个端口打上vlan 10 tag。这种端口的存在就是为了多个vlan的跨越交换机进行传递。

4. 将端口1加入104
```
interface Ten-GigabitEthernet 1/0/5 #进入三层以太网子接口视图
port link-aggregation group 104 #将此端口加入104聚合口
quit #退出
```
5. 将端口2加入104
```
interface Ten-GigabitEthernet 1/0/29 #进入三层以太网子接口视图
port link-aggregation group 104 #将此端口加入104聚合口
quit #退出
```
6. 查看所有聚合口配置
```
display interface Bridge-Aggregation 104 #查看104聚合端口配置
display link-aggregation verbose bridge-aggregation 104 #查看104聚合端口详细配置
display link-aggregation load-sharing mode interface bridge-aggregation 104 #查看104聚合端口负载模式
display link-aggregation load-sharing mode #显示全局采用的聚合负载分担类型
```