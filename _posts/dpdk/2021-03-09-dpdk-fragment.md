---
layout: post
title: DPDK分片与重组
date: 2021-03-09 17:12:00 +0800
category: DPDK
---
DPDK版本：[16.11.2](http://fast.dpdk.org/rel/)，默认支持对大报文的分片和组装，但编码过程中仍需要关注几个地方，否则会陷入单步调试的泥沼不能自拔。<br/>
> 注意点：<br/>
> 1. mbuf中的packet_type有可能并未被网卡填充，不能直接使用；
> 2. reassemble接口中frag_tbl和death_row，不是线程安全的，不支持并发访问；
> 3. mbuf库中只有read链表的操作，没有write链表的操作，所以需要自己实现；
> 4. fragment接口参数中必须保证分片大小为8的整数倍；
> 5. fragment前udp校验和的计算可参考[dpdk checksum](https://zhaotao19860.github.io/dpdk/2021/03/05/dpdk-checksum.html)；
> 6. fragment后ipv4校验和如果自己计算的话，需要重置掉offload标志；
> 7. fragment后要free掉原始的mbuf，否则send后会出现mbuf泄漏；

1. mbuf读取接口
```c
#支持读取mbuf链表，并将读取到的数据填充到buf中
static inline int pdns_pktmbuf_read(struct rte_mbuf *m, uint32_t *offset, uint32_t len, void *buf) {
    const void *tmp_ptr;
    tmp_ptr = rte_pktmbuf_read(m, *offset, len, buf);
    if (unlikely(tmp_ptr == NULL)) {
        return -1;
    }
    if (tmp_ptr != buf) {
        rte_memcpy(buf, tmp_ptr, len);
    }
    *offset += len;
    return 0;
}
```
2. mbuf写入接口
```c
struct pkt_filler_state {
    struct rte_mbuf *m; //保存第一个mbuf指针；
    struct rte_mbuf *cur_m; //保存当前使用的mbuf指针，当数据填满后，链接到第一个mbuf上；
    bool mbuf_is_new;  //报存当前使用mbuf是新建的？还是原有的；
    uint8_t *currentp; //保存当前指针位置，随数据填充后移；
    uint8_t *endp; //保存当前mbuf的结束位置，用于判断当前mbuf是否已经填满；
    uint8_t *l2p, *l3p, *l4p;    //保存2,3,4层头部指针；
    struct rte_mempool *mb_pool; //当数据超过一个mbuf时，用于分配新mbuf;
};
static int __fill_mbuf_chain(uint8_t *buf, uint16_t size, struct pkt_filler_state *state) {
    while (size > 0) {
        if (state->currentp + size < state->endp) {
            rte_memcpy(state->currentp, buf, size);
            state->cur_m->data_len += size;
            state->cur_m->pkt_len = state->cur_m->data_len;
            state->currentp += size;
            return 0;
        } else {
            //将当前节点的剩余空间使用干净
            int copy_size = state->endp - state->currentp;
            rte_memcpy(state->currentp, buf, copy_size);
            state->cur_m->data_len += copy_size;
            state->cur_m->pkt_len = state->cur_m->data_len;
            buf += copy_size;
            size -= copy_size;
            int ret = mbuf_chain(state->m, state->cur_m, state->mbuf_is_new);
            if (unlikely(ret != 0)) {
                return ret;
            }
            state->cur_m = state->cur_m->next;
            if (state->cur_m != NULL) {
                //当前链表中仍有未使用的mbuf，直接初始化使用
                state->cur_m->pkt_len = 0;
                state->cur_m->data_len = 0;
            } else {
                //当前链表已用完，需要新增分配
                state->cur_m = rte_pktmbuf_alloc(state->mb_pool);
                if (state->cur_m == NULL) {
                    log_err("rte_pktmbuf_alloc failed.");
                    return -1;
                }
                state->mbuf_is_new = true;
            }
            //初始化新mbuf的上下文指针
            state->currentp = rte_pktmbuf_mtod(state->cur_m, uint8_t *);
            state->endp =
                rte_pktmbuf_mtod_offset(state->cur_m, uint8_t *,
                                        state->cur_m->buf_len - rte_pktmbuf_headroom(state->cur_m));
        }
    }
    return 0;
}
/**
 * 填充mbuf，支持mbuf链表.
 *
 * @参数 buf
 *   执行待填充数据的指针.
 * @参数 size
 *   待填充数据的大小.
 * @参数 state
 *   当前mbuf的状态信息，用于保存上下文.
 * @返回值
 *   0：成功；
 *  -1: 失败，数据长度超过最大支持长度(默认为4个mbuf);
 *  -2：失败，mbuf分配失败；
 */
int fill_mbuf_chain(void *buf, uint16_t size, struct pkt_filler_state *state) {
    //当前mbuf足够存放数据，直接copy返回；
    if (state->currentp + size < state->endp) {
        rte_memcpy(state->currentp, buf, size);
        state->cur_m->data_len += size;
        state->cur_m->pkt_len = state->cur_m->data_len;
        state->currentp += size;
        return 0;
    }
    //当前mbuf不够存放数据，需要分配新的mbuf；
    return __fill_mbuf_chain((uint8_t *)buf, size, state);
}
```
3. 报文重组接口
```c
/*
 * 重组ipv4/ipv6分片报文.
 * @参数 m
 *   网卡接收报文.
 * @参数 portid
 *   收取报文的网卡编号.
 * @参数 queue
 *   收取报文的网卡队列编号.
 * @参数 forwarder
 *   forwarder线程配置信息.
 * @返回值
 *   1.指向mbuf的指针：
 *   - 非分片报文；
 *   - 分ipv4/ipv6报文；
 *   - 报文重组成功后的报文；
 *   2.NULL:
 *   - 报文重组失败，报文放入待删除队列；
 *   - 报文重组尚未完成，缺少分片；
 */
static inline struct rte_mbuf *reassemble(struct rte_mbuf *m, lcore_forward_t *forwarder,
                                          uint64_t tms) {
    struct ether_hdr *eth_hdr;
    struct rte_ip_frag_tbl *tbl;
    struct rte_ip_frag_death_row *dr;
    eth_hdr = rte_pktmbuf_mtod(m, struct ether_hdr *);
    /* if packet is IPv4 */
    if (eth_hdr->ether_type == rte_cpu_to_be_16(ETHER_TYPE_IPv4)) {
        struct ipv4_hdr *ip_hdr;
        ip_hdr = (struct ipv4_hdr *)(eth_hdr + 1);
        /* if it is a fragmented packet, then try to reassemble. */
        if (rte_ipv4_frag_pkt_is_fragmented(ip_hdr)) {
            tbl = forwarder->frag_tbl;
            dr = &forwarder->death_row;
            /* prepare mbuf: setup l2_len/l3_len. */
            m->l2_len = sizeof(*eth_hdr);
            m->l3_len = sizeof(*ip_hdr);
            /* process this fragment. */
            m = rte_ipv4_frag_reassemble_packet(tbl, dr, m, tms, ip_hdr);
        }
    } else if (eth_hdr->ether_type == rte_cpu_to_be_16(ETHER_TYPE_IPv6)) {
        /* if packet is IPv6 */
        struct ipv6_extension_fragment *frag_hdr;
        struct ipv6_hdr *ip_hdr;
        ip_hdr = (struct ipv6_hdr *)(eth_hdr + 1);
        frag_hdr = rte_ipv6_frag_get_ipv6_fragment_header(ip_hdr);
        if (frag_hdr != NULL) {
            tbl = forwarder->frag_tbl;
            dr = &forwarder->death_row;
            /* prepare mbuf: setup l2_len/l3_len. */
            m->l2_len = sizeof(*eth_hdr);
            m->l3_len = sizeof(*ip_hdr) + sizeof(*frag_hdr);
            /* process this fragment. */
            m = rte_ipv6_frag_reassemble_packet(tbl, dr, m, tms, ip_hdr, frag_hdr);
        }
    }
    return m;
}
```
4. 报文分片接口
```c
/*
 * Default byte size for the IPv6 Maximum Transfer Unit (MTU).
 * This value includes the size of IPv6 header.
 */
#define IPV4_MTU_DEFAULT 1500
#define IPV6_MTU_DEFAULT 1500
#define PDNS_BUFFER_MBUFS_SIZE 64
/**
 * IPv4/IPv6报文分片.
 *
 * @参数 m
 *   待分片报文.
 * @参数 m_table
 *   存储分片报文的指针数组.
 * @参数 len
 *   指针数组目前已用长度.
 * @参数 direct_pool
 *   MBUF池 - 此中分配的节点用于存储分片报文的ip头部信息.
 * @参数 indirect_pool
 *   MBUF池 - 此中分配的节点用于存储分片报文的数据部分的指针，与direct_pool一起实现了报文零拷贝.
 * @返回值
 *   空
 */
void fragment(struct rte_mbuf *m, struct rte_mbuf **m_table, int *len,
              struct rte_mempool *direct_pool, struct rte_mempool *indirect_pool) {
    int i;
    int frag_num;
    struct ether_hdr eh_copy;
    const struct ether_hdr *old_eh;
    //保存以太头部信息，用于分片报文
    old_eh = rte_pktmbuf_read(m, 0, sizeof(struct ether_hdr), &eh_copy);
    if (old_eh == NULL) {
        rte_pktmbuf_free(m);
        return;
    }
    /* Remove the Ethernet header and trailer from the input packet */
    rte_pktmbuf_adj(m, (uint16_t)sizeof(struct ether_hdr));

    /* if this is an IPv4 packet */
    if (old_eh->ether_type == rte_cpu_to_be_16(ETHER_TYPE_IPv4)) {
        /* if we don't need to do any fragmentation */
        if (likely(IPV4_MTU_DEFAULT >= m->pkt_len)) {
            m_table[*len] = m;
            frag_num = 1;
        } else {
            frag_num = rte_ipv4_fragment_packet(m, &m_table[*len],
                                                (uint16_t)(PDNS_BUFFER_MBUFS_SIZE - *len),
                                                IPV4_MTU_DEFAULT, direct_pool, indirect_pool);
            /* Free input packet */
            rte_pktmbuf_free(m);

            /* If we fail to fragment the packet */
            if (unlikely(frag_num < 0))
                return;
        }
    } else if (old_eh->ether_type == rte_cpu_to_be_16(ETHER_TYPE_IPv6)) {
        /* if we don't need to do any fragmentation */
        if (likely(IPV6_MTU_DEFAULT >= m->pkt_len)) {
            m_table[*len] = m;
            frag_num = 1;
        } else {
            //保证分片部分的大小为8的整数倍，即(IPV6_MTU_DEFAULT - sizeof(struct ipv6_hdr))%8 == 0；
            uint16_t new_mtu =
                (IPV6_MTU_DEFAULT - sizeof(struct ipv6_hdr)) / 8 * 8 + sizeof(struct ipv6_hdr);
            frag_num = rte_ipv6_fragment_packet(m, &m_table[*len],
                                                (uint16_t)(PDNS_BUFFER_MBUFS_SIZE - *len), new_mtu,
                                                direct_pool, indirect_pool);
            /* Free input packet */
            rte_pktmbuf_free(m);
            /* If we fail to fragment the packet */
            if (unlikely(frag_num < 0))
                return;
        }
    }
    /* else, just return the packet */
    else {
        m_table[*len] = m;
        frag_num = 1;
    }
    for (i = *len; i < *len + frag_num; i++) {
        m = m_table[i];
        struct ether_hdr *eth_hdr =
            (struct ether_hdr *)rte_pktmbuf_prepend(m, (uint16_t)sizeof(struct ether_hdr));
        if (eth_hdr == NULL) {
            rte_panic("No headroom in mbuf.\n");
        }
        m->l2_len = sizeof(struct ether_hdr);
        rte_memcpy(eth_hdr, old_eh, sizeof(struct ether_hdr));
        //重新计算ipv4头校验和
        if (frag_num > 1 && eth_hdr->ether_type == rte_cpu_to_be_16(ETHER_TYPE_IPv4)) {
            struct ipv4_hdr *ipv4_hdr =
                rte_pktmbuf_mtod_offset(m, struct ipv4_hdr *, sizeof(struct ether_hdr));
            ipv4_hdr->hdr_checksum = rte_ipv4_cksum(ipv4_hdr);
            //因为校验和自己计算，所以需要将offload标志去掉
            m->ol_flags &= ~PKT_TX_IP_CKSUM;
        }
    }
    *len += frag_num;
    return;
}
```