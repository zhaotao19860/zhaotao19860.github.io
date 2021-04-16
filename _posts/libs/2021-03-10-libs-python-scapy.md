---
layout: post
title: python scapy
date: 2021-03-10 09:35:00 +0800
category: Libs
---
[scapy](https://github.com/secdev/scapy)主要优势在于丰富的协议数据包构造。原理为基于libpcap(windows上是winpcap/npcap)提供的一个python接口。本文主要以dns协议为例，说明scapy的使用。
### 安装
##### 源码方式
```bash
git clone https://github.com/secdev/scapy.git
cd scapy
sudo ./run_scapy
```
##### 模块方式
```bash
pip install scapy
#交互模式
scapy
#代码模式
from scapy.all import *
```
##### 依赖安装(可选)
```bash
#如果有绘图/加密等需求：
pip install matplotlib pyx cryptography
```
### 查看支持的协议及函数
```python
ls() #查看所有支持的协议
ls(IP) #查看IP协议参数
ls(DNS) #查看DNS协议参数
lsc() #查看支持的函数
```
### 构造dns报文
```python
#如果只指定高层协议时，发包时会根据网卡信息自动填充低层的协议信息；
#如果某层的协议字段不指定时，会用默认值填充，一般为0；
package = IP(dst="2.2.3.67")/UDP(dport=53)/DNS(rd=1,qd=DNSQR(qname="www.jd.com")) 
```
### 显示报文内容
```python
ls(package)       #显示报文各字段类型及值及默认值
package.summary() #显示概要信息
package.show()    #格式化显示详细信息
package[DNS].show() #格式化显示指定协议层详细信息
Ether(raw(package)) #显示Ether头信息
DNS(raw(package)) #显示DNS协议信息
hexdump(package)  #以wireshark的格式显示数据包
wrpcap("dns.pcap",package) #可以保存报文，用wireshark查看
```
### 发送报文
```python
send() #发送三层报文
sendp() #发送二层报文
sr() #发送报文并接收响应，返回值包括answers/unanswerd(三层报文)
sr1() #发送报文并接收响应，返回值只包括一个answer(三层报文)
srp() #发送报文并接收响应，返回值包括answers/unanswerd(三层报文)
srp1() #发送报文并接收响应，返回值只包括一个answer(三层报文)
```
### 实例
#### ipv4+dns
```python
from scapy.all import *
answer = sr1(IP(dst="2.2.3.67")/UDP(dport=53)/DNS(rd=1,qd=DNSQR(qname="www.jd.com")),verbose=0)
answer.show()
print answer[DNS].summary()
answer = srp1(Ether() / IP(dst="8.8.4.4") /
        UDP(sport=RandShort(),dport=53) /
        DNS(rd=1,qd=DNSQR(qname="google.com",qclass="IN"),
            ar=DNSRROPT(rclass=3000, rdata = [EDNS0TLV(optcode=1)] )),
        timeout=1)
answer.summary()
```
#### ipv6+dns
```python
from scapy.all import *
answer1 = sr1(IPv6(dst="2001::67")/UDP(sport=RandShort(),dport=53)/DNS(rd=1,id=123,qd=DNSQR(qname="www.jd.com")))
answer2 = sr1(IPv6(dst="2001::67")/IPv6ExtHdrDestOpt()/UDP(dport=53)/DNS(rd=1,qd=DNSQR(qname="www.jd.com")))

print answer1[DNS].summary()
print answer2[DNS].summary()
```
#### 抓包
```python
p = sniff(filter="host 2001::67") #抓包
wrpcap("./ipv6.pcap",p) #保存
a=rdpcap("./ipv6.pcap") #读取
pkts = sniff(offline='ipv6.pcap') #读取
```

