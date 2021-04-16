---
layout: post
title: dpdk debug
date: 2021-02-15 11:24:00 +0800
category: DPDK
---

#### 简述
　　版本：[19.11.5](http://fast.dpdk.org/rel/)，当引用dpdk相关功能出错时，需要dpdk提供详细的出错信息或gdb调试的能力，以[librte_ip_frag](https://doc.dpdk.org/guides-19.05/sample_app_ug/ip_frag.html)库为例。

#### 步骤

1. 以调试选项编译 DPDK, 方法是在编译前执行

   ```
   export EXTRA_CFLAGS=`-g -O0`
   ```

2. 修改 /config/common_base, 将 librte_ip_frag 附近的配置修改如下:

   ```
   CONFIG_RTE_LIBRTE_IP_FRAG=y          //是否编译分片库
   CONFIG_RTE_LIBRTE_IP_FRAG_DEBUG=y    //是否打印调试日志
   CONFIG_RTE_LIBRTE_IP_FRAG_MAX_FRAG=4 //最大支持的分片数目
   CONFIG_RTE_LIBRTE_IP_FRAG_TBL_STAT=y //是否支持分片统计
   ```

3. DPDK IP分片重组库默认使用 RTE_LOGTYPE_USER1 做为调试日志, 所以用到它的程序, 如ip_reassembly 最好使用另一个 log type, 如 RTE_LOGTYPE_USER2, 方法是修改 main.c 中的宏 #define RTE_LOGTYPE_IP_RSMBL RTE_LOGTYPE_USER2.

   还需要把日志级别修改为 DEBUG, 方法是在 main 函数里 EAL 初始化后调用:

   ```
   rte_log_set_level(RTE_LOGTYPE_USER1, RTE_LOG_DEBUG);
   rte_log_set_level(RTE_LOGTYPE_USER2, RTE_LOG_DEBUG);
   ```