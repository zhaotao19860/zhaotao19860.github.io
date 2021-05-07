---
title: dns三大系统之 - 递归解析器
date: 2021-05-07 17:12:00 +0800
category: DNS
---

**Recursive Resolver** - dns递归解析器，是处于存根解析器与授权服务器之间，集缓存与递归功能于一身的dns关键模块。常用软件有bind/unbond/powerdns等，本文以bind为例简述简述递归处理逻辑。

#### 流程图(bind9)
![dns-recursive.svg](/assets/images/dns/dns-recursive.svg)

#### 参考
1. [BIND9 internals](https://ftp.cc.uoc.gr/mirrors/isc/kb-files/BIND9%20internals.pdf)