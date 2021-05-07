---
title: dns软件之 - bind9架构
date: 2021-05-07 18:07:00 +0800
category: DNS
---

**BIND9** - 作为通用性最强的dns开源软件，其设计也有许多可以参考和借鉴的地方。其采用基于事件的处理框架(libuv)，支持单线程和多线程方式。多线程中线程类型分为四种：主线程(负责初始化/信号/命令行等管理类任务)/timer/watcher/worker。bind代码可分为多个模块，每个模块包含自己的对象及对象的处理逻辑。按层次可分为：object ---> task ---> event(包含方法及数据)。object会产生event，并将event挂到指定object的task队列上，woreker线程会轮询task队列进行处理。

#### 启动流程
![bind-start.png](/assets/images/dns/bind-start.png)
&nbsp;&nbsp;&nbsp;&nbsp;原文：http://blog.also777.com/2018/05/05/bind-source-code-analysis-1/

#### 架构图
![bind9-arch.png](/assets/images/dns/bind9-arch.png)
&nbsp;&nbsp;&nbsp;&nbsp;原文：https://www.usenix.org/legacy/event/usenix06/tech/full_papers/jinmei/jinmei_html/index.html <br/>
> 说明：<br/>
timers  - 负责定时器类任务；<br/>
watcher - 负责socketio事件；<br/>
worker  - 负责其他所有任务(真正的收发包、缓存匹配、zone匹配、递归等)，与cpu个数相同；<br/>
client objects - 用于处理用户请求及递归请求；<br/>
zone table - 作为授权服务器时，管理的本地zone文件；<br/>
cache database - 作为递归和转发服务器时，管理的外部请求的缓存；<br/>


#### 参考
1. [BIND9 internals](https://ftp.cc.uoc.gr/mirrors/isc/kb-files/BIND9%20internals.pdf)
2. [Implementation and Evaluation of Moderate Parallelismin the BIND9 DNS Server](https://www.usenix.org/legacy/event/usenix06/tech/full_papers/jinmei/jinmei_html/index.html)