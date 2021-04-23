---
title: vscode远程调试
date: 2021-02-25 9:45:00 +0800
category: Tools
---
当开发linux应用时，一般的开发环境配置为：windows上完成编码，linux上完成编译加调试。但此种配置时，需要频繁的windows<=====>linux之间切换，对工作效率影响极大。[vscode](https://code.visualstudio.com/)的[debug](https://code.visualstudio.com/docs/editor/debugging)特性完美的解决了这个问题，同时支持launch(启动调试)和attach(挂载调试)。

### 要求
> vscode >= 1.35
### 安装vscode
> <https://code.visualstudio.com/>
### 安装插件
> [Remote Development](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack) <br/>
[C/C++](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools)
### server端安装openssh-server
```bash
yum install openssh-server
```
### client端安装openssh-client
> win版本>=win10：自带,不需要安装；<br/>
win版本<win10：安装git,包含ssh-client(https://git-scm.com/download)
### 配置Linux无密码登录
```bash
#生成公私钥
ssh-keygen  -C "your_email@example.com"       #-C用于设置注释
#将公钥拷贝到服务器
ssh-copy-id  your_key.pub root@YOUR-SERVER-IP 
```
### 配置 - .vscode/launch.json
```json
{
    // 使用 IntelliSense 了解相关属性。 
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            // 启动进程并调试
            "name": "Launch",
            "type": "cppdbg",
            "request": "launch",
            // 程序启动路径
            "program": "${workspaceFolder}/output/bin/adns",
            // 程序启动参数
            "args": [
                "-f",
                "${workspaceFolder}/output/etc/adns.conf"
            ],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            // 环境变量设置
            "environment": [
                {
                    "name": "LD_LIBRARY_PATH",
                    "value": "/usr/local/lib/:$LD_LIBRARY_PATH"
                }
            ],
            "externalConsole": false,
            "MIMode": "gdb",
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ]
        },
        {
            // 挂载已启动的进程
            "name": "Attach",
            "type": "cppdbg",
            "request": "attach",
            "program": "/root/zhaotao/adns-67/output/bin/adns",
            // 选择挂载的pid
            "processId": "${command:pickRemoteProcess}",
            "pipeTransport": {
                "pipeCwd": "",
                "pipeProgram": "/usr/bin/bash",
                "pipeArgs": [
                    "-c"
                ],
                "debuggerPath": "/usr/bin/gdb"
            },
            "linux": {
                "MIMode": "gdb",
                "miDebuggerPath": "/usr/bin/gdb"
            },
            // 指定源码路径
            //"sourceFileMap": {
            //    "/root/zhaotao/adns-67/src": "${workspaceFolder}/src"
            //}
        }
    ]
}
```