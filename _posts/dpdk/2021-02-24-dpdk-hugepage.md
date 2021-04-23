---
title: dpdk大页内存
date: 2021-02-24 19:59:00 +0800
category: DPDK
---
DPDK使用大页内存技术来提高系统性能，因为大页内存有以下两个特点：<br/>
> 1. hugepage 这种页面不受虚拟内存管理影响，不会被替换出内存。<br/>
1. 同样的内存大小，hugepage产生的页表项数目远少于4kpage，这样就有效的增加了TLB的命中率。

### 预留大页
#### 2M
##### 动态(非NUMA)
```bash
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
```
##### 动态(NUMA)
```bash
echo 1024 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
echo 1024 > /sys/devices/system/node/node1/hugepages/hugepages-2048kB/nr_hugepages
```
##### 静态(NUMA时均分)
```bash
#修改启动配置
sed -i -e "/GRUB_CMDLINE_LINUX/ s/\"$/ default_hugepagesz=2M hugepagesz=2M hugepages=1024\"/" /etc/default/grub
#重新生成grub
grub_rebuild
#重启
reboot
#查看是否生效
cat /proc/cmdline
#查看大小
cat /proc/meminfo |grep Huge
```
#### 1G
##### 动态
```bash
不支持
```
##### 静态
```bash
#修改启动配置
sed -i -e "/GRUB_CMDLINE_LINUX/ s/\"$/ default_hugepagesz=1G hugepagesz=1G hugepages=64\"/" /etc/default/grub
#重新生成grub
grub_rebuild
#重启
reboot
#查看是否生效
cat /proc/cmdline
#查看大小
cat /proc/meminfo |grep Huge
```
### 使用大页
#### 默认
如果使用的是默认大页，即配置中的default_hugepagesz，则dpdk使用默认挂载点/dev/hugepages。
```
不需要专门配置。
```
#### 非默认
如果系统默认是2M，用户想要使用1G，则需要手动配置。
##### 动态
```
mkdir /mnt/huge
mount -t hugetlbfs pagesize=1GB /mnt/huge
```
##### 静态
```
echo "nodev /mnt/huge hugetlbfs pagesize=1GB 0 0" >> /etc/fstab
```
### 问题
当使用的dpdk版本<18.05时，如果停止dpdk或者dpdk程序crash掉，dpdk不会回收使用的大页内存，而是第二次启动的时候重用。这就会导致其他使用大页内存的程序启动失败，解决方法如下：
```bash
1.查看大页内存挂载点
mount | grep huge
挂载点一般有两种：/dev/hugepages(默认) /mnt/huge(自定义)
2.删除挂载点下的rtemap_*文件
rm -rf rtemap_*
```