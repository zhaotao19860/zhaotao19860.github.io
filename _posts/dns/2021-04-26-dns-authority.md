---
title: dns三大系统之 - 授权
date: 2021-04-26 10:50:00 +0800
category: DNS
---

**Authority DNS**(缩写ADNS) - dns授权服务器，是dns系统中三大系统(存根解析器、递归服务器、授权服务器)中最重要的一个，负责zone文件的管理及响应用户的请求。<br/>
授权服务器处理主要分四步：zone匹配、域名匹配、类型匹配、视图匹配；<br/>
返回结果分三类：<br/>
1.如果请求域名属于本授权服务器管理，则直接返回请求记录；<br/>
2.如果请求域名属于本授权的子授权服务器，则返回子授权的NS记录；<br/>
3.其他情况，返回refuse。

#### 流程图(概略版)
![adns-flowchart-summary.png](/assets/images/dns/adns-flowchart-summary.png)

#### 流程图(详尽版)
![adns-flowchart-detail.png](/assets/images/dns/adns-flowchart-detail.png)