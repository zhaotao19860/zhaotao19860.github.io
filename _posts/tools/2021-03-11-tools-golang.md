---
layout: post
title: golang安装及vscode配置
date: 2021-03-11 09:35:00 +0800
category: Tools
---
[golang](https://golang.org/)是google推出的一种新型编程语言，拥有语法简单(类c，上手快)，静态强类型(不易引入bug)，编译型(执行速度快)，并发型(充分利用多核)，垃圾回收(减少对内存维护的代价)等优秀特性。本文主要说明安装及vscode配置相关内容。
### 安装（极其简单）
```bash
# 参考文档：https://golang.org/doc/install
# 尽量安装最新版本，避免使用插件中的各种问题
wget https://golang.org/dl/go1.16.1.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.16.1.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
go version
```
### go环境变量
```bash
# 参考文档：https://goproxy.io/zh/
# 设置代理服务器，避免下载插件失败
go env -w GOPROXY=https://goproxy.io,direct
# 还可以设置不走 proxy 的私有仓库或组，多个用逗号相隔（可选）
go env -w GOPRIVATE=git.jd.com
```
### vscode安装
```bash
# 参考文档
https://code.visualstudio.com/Download
```
### 插件安装
```bash
#参考文档
https://github.com/golang/vscode-go/blob/master/docs/tools.md
go及其依赖(Ctrl+Shift+P--->go install--->Update Tools--->全选)
```
### 代码编辑
```bash
git clone git@git.jd.com:dns-anti/common.git
ln -s `pwd`/common $GOPATH/src/git.jd.com/dns-anti/common
```
### 调试
```json
//使用dlv插件，已在插件安装部分安装。
launch.json
{
    // 使用 IntelliSense 了解相关属性。 
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Attach to Process",
            "type": "go",
            "request": "attach",
            "mode": "local",
            "processId": 36750
        },
        {
            "name": "Launch",
            "type": "go",
            "request": "launch",
            "mode": "exec",
            "program": "/export/go/src/jd.com/tp/tp-advanced-ddos/output/bin/advanced-ddos",
            "env": {},
            "args": []
        }
    ]
}
```
### 问题
1.不能正常跳转
```
vscode ---> Settings ---> 搜索Use Language Server ---> 去掉勾选；
vscode ---> Settings ---> 搜索godoc ---> 尝试选择gogetdoc/guru;
```
2.执行 go mod tiny 报错
<br/>错误:
```
https连接失败
```
原因:
```
如果使用https协议，则需要输入密码，而自动更新又没有输入框。
```
解决方案:
```bash
#修改为使用git协议，前提是配置了公私钥
git config --global url."git@git.jd.com:".insteadOf "https://git.jd.com/"
```
3.执行 go mod tiny 报错
<br/>错误:
```
go: verifying github.com/docker/docker@v1.13.1: checksum mismatch
```
原因:
```
go版本升级导致，需要重新生成go.sum
```
解决方案:
```
go clean -modcache
rm go.sum
go mode tidy
```

