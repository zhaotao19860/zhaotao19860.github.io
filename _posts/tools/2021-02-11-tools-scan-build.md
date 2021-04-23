---
title: scan-build
date: 2021-02-11 11:23:00 +0800
category: Tools
---

用于编译时静态代码调用链检测，属于clang工具箱，需要注意的是没有编译的文件是不会检查的。

## 安装
```bash
yum install -y centos-release-scl
yum install llvm-toolset-7-clang-analyzer.noarch
```

## 使用
```bash
source /opt/rh/llvm-toolset-7/enable
scan-build make
```

## 输出
```bash
启动服务端：
scan-view --host 10.226.133.67 --port 80 --allow-all-hosts --debug /tmp/scan-build-2019-12-30-102229-97575-1
web端查看：
http://10.226.133.67/
```

## 实例

![scan-build.png](/assets/images/scan-build.png)

