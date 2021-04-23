---
title: dpdk适配mellanox网卡
date: 2021-03-12 18:43:00 +0800
category: DPDK
---
以网卡Mellanox Technologies MT27710 Family [ConnectX-4 Lx]为例。

#### dpdk编译参数
由于mlx5有额外的依赖，dpdk默认不编译mlx5，因此需要修改common_base，如下
```
CONFIG_RTE_LIBRTE_MLX5_PMD=y
```
#### 安装MLNX_OFED
由于mellanox的网卡有额外依赖，因此需要安装额外的软件包，详情见dpdk官方文档：
```
https://doc.dpdk.org/guides/nics/mlx5.html#prerequisites
```
mellanox官方提供了软件包（根据操作系统，架构等选择）：
```
http://www.mellanox.com/page/products_dyn?product_family=26&mtag=linux
```
安装方法：
```
解压后执行 ./mlnxofedinstall --upstream-libs --dpdk
安装完后重启驱动：/etc/init.d/openibd restart
```
#### 网卡绑定
mellanox网卡无需使用uio，igb_uio，因此不再bind网卡到igb_uio驱动

