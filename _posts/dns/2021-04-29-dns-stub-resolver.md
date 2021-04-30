---
title: dns三大系统之 - 存根解析器
date: 2021-04-29 11:10:00 +0800
category: DNS
---

**Stub Resolver** - dns存根解析器，即dns client，是发起dns请求的起始点。用户有三种选择：1.可自己实现dns解析库；2.可以调用系统提供的库(如，gethostbyname/getaddrinfo-glibc同步dns库)；3.也可以引入第三方实现的通用库(如，bind/[adns](http://www.chiark.greenend.org.uk/~ian/adns/)-异步dns库)。<br/>
本文以chromium为例，说明存根解析器的解析流程。chromium功能比较全面，根据配置不同，支持3种client：1.可以使用自己实现的异步dns解析库；2.也支持调用系统实现的同步接口getaddrinfo；3.新版本另外增加了对doh(dns over https)的支持。

#### 流程图(chromium)
[阅读chromium源码请点击这里](https://source.chromium.org/chromium/chromium/src/+/master:net/dns/host_resolver.cc;bpv=1;bpt=1)<br/>
![dns-chromium-self.svg](/assets/images/dns/dns-chromium-self.svg)

#### 流程图(getaddrinfo)
[阅读getaddrinfo源码请点击这里](https://github.com/lattera/glibc/blob/master/sysdeps/posix/getaddrinfo.c)<br/>
![getaddrinfo.svg](/assets/images/dns/getaddrinfo.svg)

#### 他山之石(chromium)
[Chromium DNS流程分析](https://blog.finaltheory.me/research/Chromium-DNS.html)<br/>
![dns-chromium.svg](/assets/images/dns/dns-chromium.svg)