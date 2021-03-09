---
layout: post
title: dpdk checksum
date: 2021-03-05 11:25:00 +0800
category: DPDK
---

#### 简述
　　版本：[16.11.2](http://fast.dpdk.org/rel/)，当解析或组装ip/icmp/icmpv6/igmp/udp/tcp/sctp报文时，会发现都存在两个字节(16位)的checksum字段，其生成及校验算法相同。实现方式有两种：1，软件方式计算；2，offload到网卡计算。本文主要介绍软件计算方法及dpdk中的实现。<br/>
　　***注意：***当代码中设置了offload标志后，就不能使用软件方法计算，否则校验和重复计算会导致错误，因为硬件计算时校验和字段不为0。

#### 算法
```
1. 将校验和字段置为0。
2. 将每两个字节（16位）相加（二进制求和）直到最后得出结果，若出现最后还剩一个字节继续与前面结果相加。
3. (溢出)将高16位与低16位相加，直到高16位为0为止。
4. 将最后的结果（二进制）取反。
```
#### 实现
```c
变量：uint32_t cksum; //采用32位的值，用于保存16位加法的溢出位;
求和：cksum = rte_raw_cksum(const void *buf, size_t len)
求反：cksum = (~cksum) & 0xffff;
返回：return (uint16_t)cksum;
```
#### 单mbuf接口
>ipv4
```c
rte_ipv4_udptcp_cksum()
```
ipv6
```c
rte_ipv6_udptcp_cksum()
```

#### mbuf链表接口
>ipv4
```c
inline uint16_t mbuf_ipv4_udptcp_cksum(const struct rte_mbuf *m, const struct ipv4_hdr *ipv4_hdr) {
    uint32_t cksum;
    uint16_t l4_cksum;
    uint32_t l4_len;
    l4_len = rte_be_to_cpu_16(ipv4_hdr->total_length) - sizeof(struct ipv4_hdr);
    rte_raw_cksum_mbuf(m, sizeof(struct ether_hdr) + sizeof(struct ipv4_hdr), l4_len, &l4_cksum);
    cksum = l4_cksum;
    cksum += rte_ipv4_phdr_cksum(ipv4_hdr, 0);
    cksum = ((cksum & 0xffff0000) >> 16) + (cksum & 0xffff);
    cksum = (~cksum) & 0xffff;
    if (cksum == 0)
        cksum = 0xffff;
    return cksum;
}
```
ipv6
```c
inline uint16_t mbuf_ipv6_udptcp_cksum(const struct rte_mbuf *m, const struct ipv6_hdr *ipv6_hdr) {
    uint32_t cksum;
    uint16_t l4_cksum;
    uint32_t l4_len;
    //因为mbuf中不存在扩展头，所以offset的计算中不需要考虑扩展头
    l4_len = rte_be_to_cpu_16(ipv6_hdr->payload_len);
    rte_raw_cksum_mbuf(m, sizeof(struct ether_hdr) + sizeof(struct ipv6_hdr), l4_len, &l4_cksum);
    cksum = l4_cksum;
    cksum += rte_ipv6_phdr_cksum(ipv6_hdr, 0);
    cksum = ((cksum & 0xffff0000) >> 16) + (cksum & 0xffff);
    cksum = (~cksum) & 0xffff;
    if (cksum == 0)
        cksum = 0xffff;
    return cksum;
}
```