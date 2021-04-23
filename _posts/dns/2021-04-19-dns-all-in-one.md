---
title: dns全流程
date: 2021-04-20 17:28:00 +0800
category: DNS
---

**dns**(domain name server)域名服务，整个解析流程涉及3种服务器，存根解析器/递归解析器/授权服务器。

#### 相关概念
![dns_server.jpg](/assets/images/dns/dns_server.jpg)<br/>
以下分别通过dig/bind/adns三个软件模拟一个完整的递归流程。
> 名词解释：<br/>
> stub_resolver: 存根解析器，包括操作系统提供的dns解析库(getaddrinfo)或者用户自己实现的dns解析库(bind)，和相关的配置(resolv.conf/hosts)。<br/>
> recursive_resolver: 递归解析器，接收用户dns请求，完成整个递归解析流程，并将最终结果返回给存根解析器，如bind/unbound等。<br/>
> authority_server: 授权服务器，维护和保存授权记录信息，并答复用户的dns请求，如bind。<br/>
> dig: 依赖bind提供的dns解析库和resolv.conf配置文件，完成dns请求的组装、发送和dns响应的接收、解析。<br/>
> bind: 开源的dns缓存、转发、递归、授权服务器实现。<br/>
> adns: 个人实现的基于dpdk的高速dns授权服务器。<br/>

#### 拓扑结构
![all-in-one.png](/assets/images/dns/all-in-one.png)

#### 服务器配置
##### bind100
>cat  /etc/bind/named.conf

```
key "rndc-key" {
        algorithm hmac-sha256;
        secret "Sxx2b+JHmmYjTgsB/kK2BdF/1l9GG6ERs8G7k30QOBw=";
};

controls {
        inet 127.0.0.1 port 953
        allow { 127.0.0.1; } keys { "rndc-key"; };
};

options {
        directory "/etc/bind/data";
        pid-file "/etc/bind/named.pid";
        allow-query { any; };
        recursion yes;
        notify yes;
};

acl "tom" {
        localhost;
        10.226.133.100;
};

view netcom {
    match-clients { tom; };
    zone "." IN {
        type hint;
        file "db.ca";
     };
};

logging{
        channel named.log {
                file "/etc/bind/log/named.log" versions 5 size 20m;
                severity info;
                print-time yes;
                print-severity yes;
                print-category yes;
        };
        category default { named.log; };
        category lame-servers { null; };
};
```
>cat /etc/bind/data/db.ca

```
; <<>> DiG 9.12.2-P2 <<>>
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 52279
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;.                              IN      NS

;; ANSWER SECTION:
.                       435660  IN      NS      a.root-servers.net.

;; ADDITIONAL SECTION:
a.root-servers.net.     173798  IN      A       2.2.223.98
a.root-servers.net.     173798  IN      AAAA    2002::98

;; Query time: 0 msec
;; SERVER: 10.226.146.245#53(10.226.146.245)
;; WHEN: Tue Oct 09 19:51:28 CST 2018
;; MSG SIZE  rcvd: 811
```
##### adns98
>cat root-servers.net

```
. IN SOA 600 a.root-servers.net. mail.root-servers.net. 2017091705 600 900 86400 600
. DEFAULT IN NS 600 a.root-servers.net.
com. DEFAULT IN NS 600 ns1.com.
ns1.com. DEFAULT IN A 600 2.2.223.99
ns1.com. DEFAULT IN AAAA 600 2002::99
```
##### adns99
>cat com

```
com. IN SOA 600 ns1.com. mail.com. 2017091705 600 900 86400 600
com. DEFAULT IN NS 600 ns1.com.
jd.com. DEFAULT IN NS 600 ns1.jd.com.
ns1.jd.com. DEFAULT IN A 600 2.2.223.103
ns1.jd.com. DEFAULT IN AAAA 600 2002::103
```
##### adns103
>cat jd.com

```
jd.com. IN SOA 600 ns1.jd.com. mail.jd.com. 2017091705 600 900 86400 600
jd.com. DEFAULT IN NS 600 ns1.jd.com.
www.jd.com. DEFAULT IN A 600 6.6.6.6
www.jd.com. DEFAULT IN AAAA 600 6::6
```