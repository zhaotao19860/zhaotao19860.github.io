---
title: DNS Flag Day
date: 2021-05-11 14:37:00 +0800
category: DNS
---
由于线上运行的dns系统实现较多，且对标准的支持程度差别较大，导致各系统交互时，引入不必要的兼容性要求，使整个DNS系统代码变得复杂且效率降低。为了解决这个问题，由国外的一些大型的dns服务商牵头，包括ISC(bind), CZ.NIC(knot), Google, Powerdns, 思科，cloudflare等，发起了向标准协议对齐，对非标准的实现不再兼容的倡导，此倡导被称为DNS Flag Day.

### DNS Flag Day 2019
[2019.2.1](https://dnsflagday.net/2019/)为第一次共识的截止时间，此次共识主要解决授权服务器对edns的支持不符合协议引入的两个问题：
1. 如果携带edns的请求timeout，则不会重试去掉edns的请求，直接将ns标识为不可用(重试引入时延)。
2. 如果不支持edns，对于后面dns新特性的支持将受限，比如dnssec。
> 主要包括三个协议的对齐:<br/>
[RFC1035](https://datatracker.ietf.org/doc/html/rfc1035) - DNS基础协议；<br/>
[RFC2671](https://datatracker.ietf.org/doc/html/rfc2671) - EDNS0定义；<br/>
[RFC6891](https://datatracker.ietf.org/doc/html/rfc6891) - EDNS0更新；<br/>

#### 逻辑变更
主要包括以下两个方面：
1. dns-recursor对于某些服务器的EDNS查询超时，不会像以前一样再次发送不携带EDNS的查询，而是认为该服务器上的53服务已经终止。这将导致递归服务器后续的请求不会到达这台DNS服务器，导致同zone的其他NS负载上升。
2. 按照RFC6891-section7，未包含opt的请求，响应中不得包含opt；如果请求者支持opt，权威自己不支持opt，则权威应该回复formerr，并且不带opt；如果请求者支持opt，权威自己也支持opt，但是请求的opt记录中格式或者值错误，则回复formerr并携带opt记录。

#### 递归服务器
以下开源递归服务器的新版本将不提供兼容：
+ BIND 9.13.3 (development) and 9.14.0 (production)
+ Knot Resolver has already implemented stricter EDNS handling in all current versions
+ PowerDNS Recursor 4.2.0
+ Unbound 1.9.0

#### 授权服务器
授权服务器的开源实现，基本都是遵循协议的，比如Bind/Knot dns/Powerdns等，当各DNS服务提供商基本都有自己实现的版本，以下服务商已明确支持：
+ [Akamai](https://community.akamai.com/customers/s/article/CloudSecurityDNSFlagDayandAkamai20190115151216?language=en_US)
+ [Citrix](https://support.citrix.com/article/CTX241493)
+ [BlueCat](https://www.bluecatnetworks.com/blog/dns-flag-day-is-coming-and-bluecat-is-ready/)
+ [efficient iP](http://www.efficientip.com/dns-flag-day-notes/)
+ [F5 BIG-IP](https://support.f5.com/csp/article/K07808381?sf206085287=1)
+ [Google](https://groups.google.com/g/public-dns-announce/c/-qaRKDV9InA/m/CsX-2fJpBAAJ?pli=1)
+ Juniper: Older versions of the Juniper SRX will drop EDNS packets by default. The workaround is to disable DNS doctoring via # set security alg dns doctoring none. Upgrade to latest versions for EDNS support.
+ [Infoblox](https://blogs.infoblox.com/community/dns-flag-day/?es_p=8449211)
+ [Microsoft Azure](https://azure.microsoft.com/en-us/updates/azure-dns-flag-day/), [Microsoft DNS](https://support.microsoft.com/en-us/topic/windows-server-domain-name-system-dns-flag-day-compliance-85d14b63-5907-1ac2-d254-fcb1755597db)
+ [Pulse Secure](https://kb.pulsesecure.net/articles/Pulse_Secure_Article/KB43996)

#### 测试方法
1. web测试方法 - https://ednscomp.isc.org/ednscomp；
2. 源码测试方法 - https://gitlab.isc.org/isc-projects/DNS-Compliance-Testing；
3. 批量测试方法 - https://gitlab.nic.cz/knot/edns-zone-scanner/-/tree/master；
4. dig测试方法 - https://kb.isc.org/docs/edns-compatibility-dig-queries；

### DNS Flag Day 2020
[2020.10.1](https://dnsflagday.net/2020/index.html)为第二次共识的截止时间，此次共识主要解决授权服务器及递归服务器对tcp协议的支持，避免udp报文的分片发生。
> 主要包括三个协议的对齐:<br/>
[RFC7766](https://datatracker.ietf.org/doc/html/rfc7766) - DNS必须支持TCP；<br/>
[RFC6891 section 6.2.3.](https://tools.ietf.org/html/rfc6891#section-6.2.3) - 请求EDNS0 bufsize默认为1232；<br/>
[RFC6891 section 6.2.4.](https://tools.ietf.org/html/rfc6891#section-6.2.4) - 响应EDNS0 bufsize默认为1232；<br/>

#### 逻辑变更
主要包括以下两个方面：
1. 请求及响应中edns0 bufsize默认设置为1232，减少分片的可能性。
2. 对于超过1232的报文，响应中需要置截断标识位，并且客户端需要发起tcp请求，服务端需要能够处理tcp请求。

#### 递归服务器
1. 支持tcp53端口的dns请求；
2. 支持1232字节大小的bufsize，对于超过的截断。

#### 授权服务器
1. 支持tcp53端口的dns请求；
2. 支持1232字节大小的bufsize，对于超过的截断。

#### 测试方法
1. 授权web测试方法 - https://dnsflagday.net/2020/index.html#action-authoritative-dns-operators；
2. 递归web测试方法 - https://dnsflagday.net/2020/index.html#action-dns-resolver-operators；
3. dig测试方法 - https://dnsflagday.net/2020/index.html#how-to-test；

#### 服务器配置方法
下面的开源软件对tcp的支持是默认的，不需要配置，所以只配置默认的edns bufsize的大小。
+ BIND
```
options {
  edns-udp-size 1232;
  max-udp-size 1232;
};
```
+ Knot DNS
```
server:
  max-udp-payload: 1232
```
+ Knot Resolver
```
net.bufsize(1232)
```
+ PowerDNS Authoritative
```
udp-truncation-threshold=1232
```
+ PowerDNS Recursor
```
edns-outgoing-bufsize=1232
udp-truncation-threshold=1232
```
+ Unbound
```
server:
  edns-buffer-size: 1232
```
+ NSD
```
server:
  ipv4-edns-size: 1232
  ipv6-edns-size: 1232
```