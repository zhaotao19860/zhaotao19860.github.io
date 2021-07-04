---
title: dpdk交叉编译
date: 2021-07-04 15:32:00 +0800
category: DPDK
---
由于DPDK的优化中，使用特定cpu指令集获取更高的并发性能，作为其特性之一，比如SSE(Stream SIMD Extensions)，用于完成向量化操作，提升性能。这就会导致：如果编译环境使用的cpu与运行环境不同，就会出现Illegal Instruction的coredump问题。如何解决呢？交叉编译。
#### 编译参数
```
make config T=x86_64-native-linuxapp-gcc && make -j
```
说明：T = ARCH-MACHINE-EXECENV-TOOLCHAIN，针对不同的cpu架构，可以通过修改第二个参数MACHINE，完成交叉编译，默认为native(自动探测本机架构)。
#### 配置文件 - 编译参数
```
cat config/defconfig_x86_64-native-linuxapp-gcc

#include "common_linuxapp"

CONFIG_RTE_MACHINE="native"

CONFIG_RTE_ARCH="x86_64"
CONFIG_RTE_ARCH_X86_64=y

CONFIG_RTE_TOOLCHAIN="gcc"
CONFIG_RTE_TOOLCHAIN_GCC=y
```
#### 配置文件 - 平台参数
```
cat mk/machine/native/rte.vars.mk
MACHINE_CFLAGS = -march=native

SSE42_SUPPORT=$(shell $(CC) -march=native -dM -E - </dev/null | grep SSE4_2)
ifeq ($(SSE42_SUPPORT),)
  CPU_SSE42_SUPPORT = $(shell grep SSE4\.2 /var/run/dmesg.boot 2>/dev/null)
  ifneq ($(CPU_SSE42_SUPPORT),)
    MACHINE_CFLAGS = -march=corei7
  endif
endif
```
#### 查看支持的平台(Code Name缩写)
```
ls mk/machine
atm  default  hsw  ivb  native  nhm  snb  wsm
```
#### 查看指定的平台的CodeName
```
#查看cpu型号
cat /proc/cpuinfo |grep "model name"|head -1
model name	: Intel(R) Xeon(R) CPU E5-2620 0 @ 2.00GHz
#登录网站查看
ark.intel.com
  Intel® Xeon® Processor E5 Family
  Code Name Products formerly Sandy Bridge EP
  Vertical Segment Server
  Processor Number E5-2620
```

缩写|型号
---|---
wsm|Westmere
nhm|Nehalem
snb|Sandy Bridge
ivb|Ivy Bridge
hsw|HasWell
#### 生成编译配置
```
cp config/defconfig_x86_64-native-linuxapp-gcc config/defconfig_x86_64-snb-linuxapp-gcc
vim config/defconfig_x86_64-snb-linuxapp-gcc
CONFIG_RTE_MACHINE="native" ===> CONFIG_RTE_MACHINE="snb"
```
#### 生成平台配置(如果存在，跳过)
```
#查看平台在gcc对应的march
gcc -c -Q -march=native --help=target | grep march
-march=core2
cp -rf mk/machine/ivb mk/machine/snb
vim mk/machine/snb/rte.vars.mk
MACHINE_CFLAGS = -march=core-avx-i ===> MACHINE_CFLAGS = -march=core2
```
#### 重新编译
```
make config T=x86_64-snb-linuxapp-gcc && make -j
```