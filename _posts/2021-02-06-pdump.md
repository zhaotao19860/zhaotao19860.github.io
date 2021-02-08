---
layout: post
title: pdump
date: 2021-02-06 10:23:00 +0800
category: DPDK
---

#### 简述

类似与tcpdump，可将dpdk的收发报文保存为pcap文件格式。
收包：网卡-->DMA-->rx_ring-->rx_burst-->rx_pkt_burst-->pdump_callback-->app;
发包：app-->tx_burst-->pdump_callback-->tx_pkt_burst-->tx_ring-->DMA-->网卡。

#### 安装及使用

```bash
1)安装依赖
  yum install -y libpcap-devel
2)修改配置
  vim dpdk-stable-19.11.5/config/common_linux
  末尾添加CONFIG_RTE_LIBRTE_PMD_PCAP=y
3)重新编译
  cd dpdk-stable-19.11.5
  export RTE_SDK=/export/servers/dpdk-stable-19.11.5
  export RTE_TARGET=x86_64-native-linuxapp-gcc
  make -j install T=x86_64-native-linuxapp-gcc(重新编译)
4)使用
  vi main.c
  添加
  #include <rte_pdump.h>
  在rte_eal_init()后面增加
  #if 1
  /* initialize packet capture framework */
  rte_pdump_init(NULL);
  #endif
  make //重新编译应用
5)启动应用程序
6)抓包
 /export/servers/dpdk-stable-19.11.5/x86_64-native-linuxapp-gcc/app/dpdk-pdump -- --pdump 'port=0,queue=*,rx-dev=./dpdk0.pcap,tx-dev=./dpdk0.pcap' --pdump 'port=1,queue=*,rx-dev=./dpdk1.pcap,tx-dev=./dpdk1.pcap'
```

#### 抓包脚本

```bash
#!/bin/bash
# Created by tom on 2019/11/21

# Exit immediately if a command exits with a non-zero status.
#set -e

SERVICE_BIN="/export/servers/dpdk-stable-16.11.2/x86_64-native-linuxapp-gcc/app/dpdk-pdump"

usage(){
    echo "===================================================================================="
    echo "使用说明："
    echo "    sh ./pdump.sh help         //显示帮助信息"
    echo "    sh ./pdump.sh start 2      //启动pdump抓取2个网卡的报文并将报文保存在/tmp/dpdk-pdump"
    echo "    sh ./pdump.sh stop         //停止抓包"
    echo "    sh ./pdump.sh status       //显示当前pdump状态"
    echo "    sh ./pdump.sh save         //保存当前抓取的所有报文到/export/pdump目录下"
    echo "===================================================================================="
}

init(){
    SERVICE="dpdk-pdump"
    SHELL_FOLDER=$(cd "$(dirname "$0")";pwd)
    cd ${SHELL_FOLDER} 
    OUT_FOLDER="/tmp/$SERVICE/"
    SAVE_FOLDER="/export/pdump/"
    FILE_NUM_LIMIT=10
    mkdir -p $OUT_FOLDER
    mkdir -p $SAVE_FOLDER
    #关闭linux内核参数 - ASLR(Address space layout randomization);
    #此参数会导致pdump不停的coredump;
    echo 0 > /proc/sys/kernel/randomize_va_space
    log_rotate
}

is_alive(){
    count=`ps -ef | grep "$SERVICE" | grep -v "grep" | wc -l`
    if [ $count -ne 0 ]
    then
        return 0
    else
        return 1
    fi
}

log_rotate(){
    count_current=`ls $OUT_FOLDER/*.log $OUT_FOLDER/*.pcap| awk -F'_' '{print $1}' |sort -n|uniq|wc -l`
    if [ $count_current -gt $FILE_NUM_LIMIT ]
    then
        ((num_to_delete=$count_current-$FILE_NUM_LIMIT))
        item_to_delete=`ls $OUT_FOLDER/*.log $OUT_FOLDER/*.pcap | awk -F'_' '{print $1}' |sort -n|uniq|head -$num_to_delete`
        for item in $item_to_delete
        do
            rm -rf ${item}*
        done
    fi
}

status(){
    if is_alive
    then
        echo "$(date) $SERVICE is running."
    else
        echo "$(date) $SERVICE is not running."
    fi
}

start(){
    if ! is_alive
    then
        echo "$(date) $SERVICE starting..."
        date_str=`date "+%Y%m%d%H%M%S"`
        for ((i=0; i<$1; i++))
        do
            pdump_para_str=$pdump_para_str" --pdump ""port=$i,queue=*,rx-dev=$OUT_FOLDER/${date_str}_port${i}_rx.pcap,tx-dev=$OUT_FOLDER/${date_str}_port${i}_tx.pcap" 
        done
        cd $OUT_FOLDER #如果有core文件，则生成到out_folder目录
        nohup $SERVICE_BIN -- $pdump_para_str >> $OUT_FOLDER/${date_str}_dpdk-pdump.log 2>&1 & 
        sleep 1
        if is_alive
        then
            echo "$(date) $SERVICE start success."
        else
            echo "$(date) $SERVICE start failed, please find the ERROR in the log [$OUT_FOLDER/${date_str}_dpdk-pdump.log]."
            return 1
        fi
    else
        echo "$(date) $SERVICE is already running."
        return 0
    fi
}

stop(){
    if is_alive
    then
        echo "$(date) $SERVICE stopping ..."
        pkill -2 "$SERVICE"
        sleep 1
        if is_alive
        then
            echo "$(date) stop(kill -2) $SERVICE failed, now use kill -9..."
            pkill -9 "$SERVICE"
            sleep 1
            if is_alive
            then
                echo "$(date) stop(kill -9) $SERVICE failed."
                return 1
            else
                echo "$(date) stop(kill -9) $SERVICE succeed."
                return 0
            fi
        else
            echo "$(date) stop(kill -2) $SERVICE succeed."
            return 0
        fi
    else
        echo "$(date) $SERVICE is already stopped."
        return 0
    fi
}

save(){
    if is_alive
    then
        stop
    fi
    date_str=`date "+%Y%m%d%H%M%S"`
    mkdir -p $SAVE_FOLDER/${date_str}
    \cp $OUT_FOLDER/*.pcap $SAVE_FOLDER/${date_str}/
    \cp $OUT_FOLDER/*.log $SAVE_FOLDER/${date_str}/
}


main(){
    init
    case "$1:$2" in
        start:[1-2]) start $2 || exit 0 ;;
        stop:) stop || exit 0 ;;
        status:) status || exit 0 ;;
        save:) save || exit 0 ;;
        *) usage ;;
    esac
}

main $@
```