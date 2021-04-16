---
layout: post
title: kswapd0
date: 2021-02-13 20:58:00 +0800
category: Linux
---

## 概述
&nbsp;&nbsp;&nbsp;&nbsp;线上某台服务器频繁告警，调用python->dircmp获取变化的文件时，返回超过20秒，增加打印日志发现是cmp函数返回慢。<br>
因为cmp函数需要比较文件内容，需要大量内存，同时top发现kswap0一直运行且cpu占用100%，但通过free命令查看内存，<br>
swap分区基本没有使用，初步判定是系统内核自动使用了swap，需要调整系统参数。

## 解决方案
```bash
1.尽量不使用缓存：
  echo vm.swappiness=0 | sudo tee -a /etc/sysctl.conf && sysctl -p
2.清理缓存
  sync && echo 3 > /proc/sys/vm/drop_caches(只会执行一次，如果需要定期执行请设置crontab)
```

