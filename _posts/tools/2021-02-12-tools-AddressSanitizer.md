---
title:  AddressSanitizer
date:   2021-02-12 19:12:00 +0900
categories: Tools
---

AddressSanitizer（地址杀菌剂，简称 ASan） 是谷歌出品的内存检查工具，可以诊断内存越界，
非法访问，内存泄漏，内存double free等常见内存问题，并且效率比valgrind高好几倍，可以克服valgrind的一些问题，比如占用内存高的问题。<br>
&nbsp;&nbsp;&nbsp;&nbsp;gcc 4.8 开始，AddressSanitizer 成为 gcc 的一部分，但不支持符号信息，无法显示出问题的函数和行数。
从 4.9 开始，gcc 支持 AddressSanitizer 的所有功能。<br>
Github 地址：https://github.com/google/sanitizers。  <br>
Wiki 地址：https://github.com/google/sanitizers/wiki/AddressSanitizer。

## 安装
```bash
yum install -y centos-release-scl 
yum install -y devtoolset-3-toolchain
yum install -y devtoolset-3-libasan-devel.x86_64
```

## 使用
```bash
1.切换gcc(4.8->4.9):
  source /opt/rh/devtoolset-3/enable
2.编译选项：
  CFLAGS += -ggdb3 -gdwarf-2 -static-libasan -fsanitize=address -fno-omit-frame-pointer
  -fsanitize=address       #开启地址越界检查功能
  -fno-omit-frame-pointer  #开启后，可以出界更详细的错误信息
  -fsanitize=leak          #开启内存泄露检查功能
3.运行选项：
  ASAN_OPTIONS=verbosity=1:malloc_context_size=20 ./a.out
4.标志说明:
  https://github.com/google/sanitizers/wiki/AddressSanitizerFlags
```