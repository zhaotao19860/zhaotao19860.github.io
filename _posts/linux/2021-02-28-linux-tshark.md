---
title: tshark
date: 2021-02-28 18:33:00 +0800
category: Linux
---

tshark是wireshark的命令行工具，同tcpdump一样，可通过libpcap抓取/过滤/保存网卡报文，处理速度低于tcpdump，但支持的协议更加丰富。
> 重点参数：<br/>
> -f : 抓包过滤(capture filter) <https://tshark.dev/capture/capture_filters/><br/>
> -Y/-R : 显示过滤(display filter) <https://tshark.dev/analyze/packet_hunting/packet_hunting/>

## 安装
```bash
yum install -y epel-release
yum install wireshark
```
## 使用
1. 实时打印dns协议
```bash
tshark -i eth0 -f "udp and port 53" -Y 'dns.qry.name==www.baidu.com' -l -T fields -e dns.qry.name -e dns.count.queries -e dns.id -e ip.src
```
>参数含义：<br/>
    -i eth0 :捕获eth0网卡<br/>
    -f 'udp and port 53' :抓包过滤：指定协议和端口<br/>
    -Y dns.qry.name=="www.baidu.com" :显示过滤：指定域名<br/>
    -l 实时输出<br/>
    -T fields -e dns.qry.name -e dns.count.queries -e dns.id -e ip.src :按列显示，请求域名/请求个数/transactionId/源IP<br/>

2. 实时打印当前http请求的url
```bash
tshark -s 512 -i eth0 -n -f 'tcp dst port 80' -R 'http.host and http.request.uri' -T fields -e http.host -e http.request.uri -l | tr -d '\t'
```
>参数含义：<br/>
    -s 512 :只抓取前512个字节数据<br/>
    -i eth0 :捕获eth0网卡<br/>
    -n :禁止网络对象名称解析<br/>
    -f 'tcp dst port 80' :只捕捉协议为tcp,目的端口为80的数据包<br/>
    -R 'http.host and http.request.uri' :过滤出http.host和http.request.uri<br/>
    -T fields -e http.host -e http.request.uri :打印http.host和http.request.uri<br/>
    -l ：实时输出

3. 实时打印当前mysql查询语句
```bash
tshark -s 512 -i eth0 -n -f 'tcp dst port 3306' -R 'mysql.query' -T fields -e mysql.query
```
>参数含义：<br/>
    -s 512 :只抓取前512个字节数据<br/>
    -i eth0 :捕获eth0网卡<br/>
    -n :禁止网络对象名称解析<br/>
    -f 'tcp dst port 3306' :抓包过滤，只捕捉协议为tcp,目的端口为3306的数据包<br/>
    -R 'mysql.query' :显示过滤，mysql.query<br/>
    -T fields -e mysql.query :打印mysql查询语句<br/>