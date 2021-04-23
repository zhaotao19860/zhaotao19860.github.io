---
title: vscode gdb
date: 2021-02-26 10:49:00 +0800
category: Tools
---

当使用gdb调试应用时，如果需要执行vscode不支持的一些gdb命令，需要用到-exec命令，如查看内存。先决条件：代码编译时增加-g或-ggdb选项，如果要包含宏定义，需要-g3/-ggdb3级别。
>-g: 该选项可以利用操作系统的“原生格式（native format）”生成调试信息。GDB 可以直接利用这个信息，其它调试器也可以使用这个调试信息。<br/>
-ggdb: 使 GCC为GDB 生成专用的更为丰富的调试信息，但是，此时就不能用其他的调试器来进行调试了 (如 ddx)。

#### 打印内存
```
-exec x/nfu addr
说明：
a）n：输出单元的个数。
b）f：是输出格式。比如x是以16进制形式输出，o是以8进制形式输出,等等。
c）u：标明一个单元的长度。b是一个byte，h是两个byte（halfword），w是四个byte（word），g是八个byte（giant word）。
例子：
打印地址a开始的16个16进制字节
-exec x/16xb a
0x7fffffffe4a0: 0x00    0x01    0x02    0x03    0x04    0x05    0x06    0x07
0x7fffffffe4a8: 0x08    0x09    0x0a    0x0b    0x0c    0x0d    0x0e    0x0f
```
#### 打印寄存器
```
-exec info registers
```
#### 显示源码路径
```
-exec list
```
#### 更换源码路径
```
-exec set substitute-path /home/jenkins/workspace/dns-anti.TPDNS.master.29553/src /tmp/TPDNS-v2.0.1-master/src
```
#### 查看断点
```
-exec info b
```
#### 查看堆栈变量
```
-exec i locals
```