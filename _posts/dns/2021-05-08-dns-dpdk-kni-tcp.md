---
title: 基于dpdk实现的dns-tcp解决方案
date: 2021-05-08 17:22:00 +0800
category: DNS
---
由于DPDK没有提供网络协议栈，所以在具体应用中，如果需要tcp协议的支持，就需要自己实现或使用kni接口将请求转入内核处理。DNS服务99%的服务是基于UDP的，tcp流量主要用于大报文(大于512字节)或主辅同步，且较少，所以可以采用kni旁路的办法。本文主要讲解大体部署方案，以供参考。

#### 基本原理
利用DPDK的flow director及KNI功能，将来自用户(vip)的tcp请求全部转给linux kernel交给BIND处理，BIND将tcp转为udp请求，转发给dns服务器，dns服务器组响应包并返回给BIND,BIND组tcp响应返回给用户。<br/>
**注意**：
```
1.根据forward ip地址的配置不同BIND forward时选择的网卡也不同(根据内核路由)，可以是vEth0网卡，也可以是其他非dpdk网卡(下面的数据流图为走vEth0的流程，走其他网卡类似)。
2.Tcp响应报文发送网卡根据内核路由确定，可以为eth0，也可以为vEth0，但为了保证能正常响应公网IP，出口网卡必须有出外网的权限；
```

#### 数据流图
![ADNS-TCP.webp](/assets/images/dns/ADNS-TCP.webp) <br/>
**说明**：<br/>
1.用户发送tcp-dns请求到ADNS1；<br/>
2.匹配fdir-tcp过滤条件；<br/>
3.Fdir-tcp对应0-kni队列；<br/>
4.KNI收包线程读取0号队列报文；<br/>
5.KNI收包线程将报文转发给内核中的Veth0虚拟网卡；<br/>
6.KNI内核线程收包并将mbuf报文转为skb内核报文，然后发给bind监听的socket;<br/>
7.Bind从socket收包并将报文转为udp报文，然后根据forward-ip发送到Veth0或者eth0;<br/>
8.协议栈内核线程将skb内核报文转为mbuf报文，然后放入KNI发送队列；<br/>
9.KNI发包线程读取发送队列报文，并放入0号发送队列；<br/>
10.网卡将udp请求报文发出；<br/>
11.源IP为ADNS1的vEth0网卡ip，目的IP为ADNS2的vEth0网卡ip,通过路由转给ADNS2;<br/>
12.匹配fdir-dstport53过滤条件;<br/>
13.Fdir-dstport53对应1号队列；<br/>
14.Worker线程接收udp dns请求并组装响应报文；<br/>
15.Worker线程将响应报文放入1号发送队列；<br/>
16.网卡将udp响应报文发出；<br/>
17.源IP为ADNS2的vEth0网卡ip，目的IP为ADNS1的vEth0网卡ip,通过路由转给ADNS1;<br/>
18.匹配fdir-dstip过滤条件;<br/>
19.Fdir-dstip对应0号队列；<br/>
20.KNI收包线程读取0号队列报文；<br/>
21.KNI收包线程将报文转发给内核中的vEth0虚拟网卡；<br/>
22.KNI内核线程收包并将mbuf报文转为skb内核报文，然后发给BIND监听的socket(如果转发端口非DPDK，走正常内核流程);<br/>
23.BIND从socket收包并将报文转为tcp报文，然后通过查找内核路由发送到vEth0或eth0;<br/>
24.协议栈内核线程将skb内核报文转为mbuf报文，然后放入KNI发送队列；<br/>
25.KNI发包线程读取发送队列报文，并放入0号发送队列；<br/>
26.网卡将tcp响应报文发送给用户；<br/>

#### bind安装及配置
##### 安装
```
1.安装bind
  yum install bind -y or ./configure && make && make install
2.配置named
  vim /etc/named.conf
  forward 改为同一vip的  另外一台实例的vEth0ip
3.添加所有vip到Lo口(保证bind可以处理目的地址是vip的请求)
  ip addr  add $vip/32 dev lo
4.增加默认路由，保证vip的tcp响应报文能正常返回
  ip route add default via 10.238.69.1(vEth0网关)
5.重启named
  systemctl  restart named
6.测试
  dig +tcp @vip jiankong.jdgslb.com
```
#### 配置
vi named.conf
```
//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//

options {
    listen-on port 53 { any; };
    listen-on-v6 port 53 { any; };
    directory     "/var/named";
    dump-file     "/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";
    allow-query     { any; };

        forward only;
        forwarders{
        100.125.255.255;
        #10.226.146.245;
        };
    /*
     - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
     - If you are building a RECURSIVE (caching) DNS server, you need to enable
       recursion.
     - If your recursive DNS server has a public IP address, you MUST enable access
       control to limit queries to your legitimate users. Failing to do so will
       cause your server to become part of large scale DNS amplification
       attacks. Implementing BCP38 within your network would greatly
       reduce such attack surface
    */
    recursion yes;

    dnssec-enable no;
    dnssec-validation no;

    /* Path to ISC DLV key */
    bindkeys-file "/etc/named.iscdlv.key";

    managed-keys-directory "/var/named/dynamic";

    pid-file "/run/named/named.pid";
    session-keyfile "/run/named/session.key";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
    type hint;
    file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```