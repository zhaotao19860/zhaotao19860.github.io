---
title: dpdk-as-lib
date: 2021-02-10 19:35:00 +0800
category: DPDK
---

当把dpdk作为第三方库使用时，makefile该怎么写呢？当自己的代码是c++的时候该怎么改呢？
以下使用dpdk-stable-16.11.2/examples/ip_reassembly为例。

#### c引用

1. 说明

   ```
   a. 必须声明RTE_SDK环境变量；
   b. 可选声明RTE_TARGET环境变量，默认为：x86_64-native-linuxapp-gcc ；
   c. 起始处包含 $(RTE_SDK)/mk/rte.vars.mk 文件。
   d. 末尾包含指定的 $(RTE_SDK)/mk/rte.XYZ.mk 文件，其中 XYZ 可以是 app(dpdk中程序)、lib(dpdk中库)、
      extapp(外部程序), extlib(外部库)等等，取决于要编译什么样的目标文件。
   e. 其他变量：
      APP - 应用名称;
      SRCS-y - 源码文件；
      CFLAGS - gcc编译选项；
   ```

2. 实例

   ```makefile
   include $(RTE_SDK)/mk/rte.vars.mk
   
   # binary name
   APP = ip_reassembly
   
   # all source are stored in SRCS-y
   SRCS-y := main.c
   
   CFLAGS += -O3
   CFLAGS += $(WERROR_FLAGS)
   
   include $(RTE_SDK)/mk/rte.extapp.mk
   ```

#### c++引用

1. 说明

   ```
   a. 大部分与c引用一样；
   b. 增加以下变量：
      CC = g++
      CXXFLAGS += -std=gnu++11
      LDFLAGS += -lstdc++
   c. SRCS-y源码路径改为绝对路径(原因不详)
   ```

2. 实例

   ```makefile
   include $(RTE_SDK)/mk/rte.vars.mk
   
   # binary name
   APP = ip_reassembly
   
   # all source are stored in SRCS-y
   SRCS-y := /test/main.c
   
   CC = g++
   CXXFLAGS += -std=gnu++11
   CFLAGS += -O3
   LDFLAGS += -lstdc++
   
   include $(RTE_SDK)/mk/rte.extapp.mk
   ```