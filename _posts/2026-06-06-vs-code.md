---
layout: post
title: "局域网远程访问本机 VS Code 环境手册"
date: 2026-06-06 13:30:51 +0800
categories: [Tools]
tags: [VS Code, SSH, Remote Tunnel, macOS]
---

# 局域网远程访问本机 VS Code 环境手册

在 Mac 上把 VS Code 暴露给同局域网的其他设备（Windows、Linux、另一台 Mac、iPad），常见有两条路：**VS Code Remote Tunnel** 走 GitHub 中继，**SSH 直连**走局域网。本文给出两种方案的最简配置、开机自启与故障排查，并给出选型建议。

---

## 方案对比

| | VS Code Tunnel | SSH |
|---|---|---|
| **原理** | 通过 GitHub 中继连接 | 局域网直连 |
| **网络要求** | 需要联网（走 GitHub 中继） | 纯局域网即可，无需外网 |
| **延迟** | 低 | 更低（直连） |
| **配置难度** | 一条命令 | 需开启 SSH 服务 |
| **IDE 扩展** | ✅ 完整支持 | ✅ 完整支持 |
| **浏览器访问** | ✅ vscode.dev | ❌ |
| **适用场景** | 跨网络 / 不想配 SSH | 局域网 / 追求最低延迟 |

---

## 一、VS Code Remote Tunnel

### 1. 本地机器：启动隧道

```bash
# 首次运行，按提示完成 GitHub 授权
code tunnel

# 看到 "Tunnel started successfully" 即成功
# 授权一次后后续无需再登录
```

### 2. 远程机器：连接

**方式 A — VS Code 客户端（推荐）**

1. 安装 VS Code 与扩展 `Remote - Tunnels`
2. 用同一个 GitHub 账号登录
3. `Cmd+Shift+P` → `Remote-Tunnels: Connect to Tunnel`
4. 选择本地机器名

**方式 B — 浏览器**

1. 访问 https://vscode.dev
2. 用同一个 GitHub 账号登录
3. `Cmd+Shift+P` → `Remote-Tunnels: Connect to Tunnel`

### 3. 设置开机自启 + 后台常驻

首次手动授权成功后，创建 launchd 服务，让隧道随系统启动并崩溃自愈。

```bash
# 确认 code 路径
which code
```

创建 `~/Library/LaunchAgents/com.microsoft.vscode.tunnel.plist`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.microsoft.vscode.tunnel</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/code</string>
        <string>tunnel</string>
        <string>--accept-server-license-terms</string>
        <string>--name</string>
        <string>my-mac-tunnel</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/tmp/vscode-tunnel.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/vscode-tunnel.err</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
        <key>HOME</key>
        <string>/Users/tom</string>
    </dict>
</dict>
</plist>
```

> **需要修改的字段**：`/usr/local/bin/code` 替换为 `which code` 的实际输出；`my-mac-tunnel` 替换为你想要的隧道名；`/Users/tom` 替换为你的实际 HOME 目录。

加载服务：

```bash
launchctl load ~/Library/LaunchAgents/com.microsoft.vscode.tunnel.plist
```

### 4. Tunnel 管理命令

```bash
# 启动
launchctl load ~/Library/LaunchAgents/com.microsoft.vscode.tunnel.plist

# 停止
launchctl unload ~/Library/LaunchAgents/com.microsoft.vscode.tunnel.plist

# 重启
launchctl unload ~/Library/LaunchAgents/com.microsoft.vscode.tunnel.plist
launchctl load ~/Library/LaunchAgents/com.microsoft.vscode.tunnel.plist

# 查看状态
launchctl list | grep vscode.tunnel

# 查看日志
tail -f /tmp/vscode-tunnel.log
```

---

## 二、SSH 直连

### 1. 本地机器：开启 SSH 服务

**系统设置 → 通用 → 共享 → 远程登录** → 打开。

或用命令行：

```bash
sudo systemsetup -setremotelogin on
```

获取本机局域网 IP：

```bash
ipconfig getifaddr en0
# 输出如 192.168.1.100
```

### 2. 远程机器：连接

**方式 A — VS Code 客户端（推荐）**

1. 安装 VS Code 与扩展 `Remote - SSH`
2. `Cmd+Shift+P` → `Remote-SSH: Connect to Host`
3. 输入 `tom@192.168.1.100`
4. 输入密码，连接成功

**方式 B — 终端 SSH**

```bash
ssh tom@192.168.1.100
```

### 3. 配置 SSH 免密登录（推荐）

在远程机器上执行：

```bash
# 生成密钥（如果还没有）
ssh-keygen -t ed25519

# 复制公钥到本地机器
ssh-copy-id tom@192.168.1.100

# 之后连接无需密码
ssh tom@192.168.1.100
```

### 4. VS Code SSH 配置文件（简化连接）

在远程机器编辑 `~/.ssh/config`：

```
Host my-mac
    HostName 192.168.1.100
    User tom
    IdentityFile ~/.ssh/id_ed25519
```

之后 VS Code 里只需选 `my-mac` 即可连接。

### 5. SSH 保持连接防断开

在远程机器的 `~/.ssh/config` 中添加：

```
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

---

## 三、故障排查速查

| 问题 | Tunnel 方案 | SSH 方案 |
|------|-----------|---------|
| 连不上 | 检查 `code tunnel` 是否运行 | 检查「远程登录」是否开启 |
| 授权失败 | 重新手动运行 `code tunnel` 登录 | 检查密码/密钥是否正确 |
| 扩展缺失 | 在远程端安装本地扩展 | 同左 |
| 连接断开 | 检查网络 | 添加 ServerAliveInterval |
| 查日志 | `tail -f /tmp/vscode-tunnel.log` | `log show --predicate 'process == "sshd"'` |

---

## 四、快速选择

- **有外网 + 想省事** → VS Code Tunnel（一条命令搞定）
- **纯局域网 + 追求低延迟** → SSH 直连
- **两个都配** → 日常用 SSH（快），外出用 Tunnel（随时连）
