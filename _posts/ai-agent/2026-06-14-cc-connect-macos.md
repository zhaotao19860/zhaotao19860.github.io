---
layout: post
title: "把 cc-connect 跑起来：从编译修复到 macOS 后台服务"
date: 2026-06-14 23:16:23 +0800
categories: [AI Agent]
tags: [cc-connect, Claude Code, Codex, macOS, launchd]
---

# 把 cc-connect 跑起来：从编译修复到 macOS 后台服务

这次折腾 `cc-connect` 的过程，刚好把一个本地 AI Agent 桥接工具从“源码能编译”推进到了“后台常驻 + Web 管理端可访问”的状态。过程里踩到的问题不复杂，但很典型：本机工具链缺失、网络代理、平台 build tag、单测夹具，以及最后的后台服务配置。

## cc-connect 是什么

`cc-connect` 的定位是把本地 AI Coding Agent（比如 Claude Code、Codex、Cursor、Gemini CLI 等）接到消息平台或 Web 管理端上。它本身不直接实现大模型能力，而是作为一层桥：

- 平台侧负责接收消息，比如飞书、微信、Telegram 等。
- Agent 侧负责调用本机已有的命令行工具。
- 中间由 `cc-connect` 管理会话、权限、日志、后台服务和管理 API。

所以它对本机环境有一个重要前提：本地 Agent 通常需要是可执行的 CLI 工具。

## 第一步：编译失败，发现本机没有 Go

一开始执行：

```bash
cd cc-connect
make build
```

前端部分顺利完成：

```text
cd web && npm run build
vite v6.4.3 building for production...
✓ built
```

但后端 Go 编译失败：

```text
/bin/sh: go: command not found
make: *** [build] Error 127
```

这说明项目本身已经进入 Go 编译阶段，但本机缺少 Go 工具链。检查 `go.mod` 后发现项目要求：

```go
go 1.25.0
```

于是使用 Homebrew 安装 Go。第一次安装因为 GitHub Container Registry 下载 bottle 时 HTTP/2 异常失败：

```text
curl: (92) HTTP/2 stream 1 was not closed cleanly: PROTOCOL_ERROR
```

重试时关闭 Homebrew 自动更新，安装成功：

```bash
HOMEBREW_NO_AUTO_UPDATE=1 brew install go
```

最终确认版本：

```text
go version go1.26.4 darwin/arm64
```

## 第二步：Go 依赖下载超时，切换 GOPROXY

安装 Go 后再次 `make build`，前端仍然成功，但 Go 模块下载卡在 `proxy.golang.org`：

```text
Get "https://proxy.golang.org/...": dial tcp ... i/o timeout
```

这是国内环境下常见的问题。解决方式是设置 Go 模块代理：

```bash
go env -w GOPROXY=https://goproxy.cn,direct
```

之后依赖下载顺利通过。

## 第三步：修复 macOS 下的 build tag 冲突

依赖下载完成后，出现了真正的代码编译错误：

```text
daemon/linger_other.go:6:6: CheckLinger redeclared in this block
	daemon/launchd.go:32:6: other declaration of CheckLinger
```

原因是 `daemon/linger_other.go` 的 build tag 是：

```go
//go:build !linux
```

这会让它在 macOS 上也参与编译，而 `daemon/launchd.go` 本身就是 macOS 专用文件，也定义了 `CheckLinger`。因此需要把 `linger_other.go` 限制为非 Linux、非 macOS、非 Windows 的兜底实现：

```go
//go:build !linux && !darwin && !windows
```

这个改动让 macOS 下只使用 `launchd.go` 中的实现，避免重复定义。

## 第四步：补齐 launchd 状态测试夹具

修复 build tag 后，执行：

```bash
go test ./daemon && make build
```

又遇到一个单测失败：

```text
--- FAIL: TestLaunchdStatusUsesUserDomainWhenGUIDomainUnavailable
    launchd_test.go:99: Status().Running = false, want true
```

排查后发现，测试模拟了 GUI domain 不可用、user domain 可用的场景，但 `Status()` 的逻辑会先检查 plist 是否存在。如果测试没有创建临时 plist，它会提前返回“未安装”，自然也就不会进入后续 launchd target 检测。

修复方式是在测试里设置临时 `HOME`，创建 `Library/LaunchAgents/com.cc-connect.service.plist`，再验证 fallback 行为。这样测试更贴近真实“已安装 LaunchAgent”的状态。

最终验证通过：

```text
ok   github.com/chenhg5/cc-connect/daemon
...
go build ... -o cc-connect ./cmd/cc-connect
```

`make build` 成功产出本地二进制：

```text
/Users/tom/Desktop/github/cc-connect/cc-connect
```

## 第五步：理解 Agent 启动方式

在使用过程中，我顺手确认了一下 `cc-connect` 对本地 Agent 的启动模型。

核心接口抽象为 `AgentSession`，强调“会话”：

- `StartSession` 创建或恢复会话。
- `Send` 向会话发送消息。
- `Events` 持续输出事件。
- `Close` 关闭底层进程或会话。

但不同 Agent 适配器的进程生命周期不完全一样。

### Claude Code：长驻进程

Claude Code 适配器不是用 `claude -p`，而是启动一个长时间运行的 `claude` 进程，并通过 stdin/stdout 进行 stream-json 交互。启动参数类似：

```bash
claude \
  --output-format stream-json \
  --input-format stream-json \
  --permission-prompt-tool stdio \
  --replay-user-messages
```

如果有模型、权限模式、恢复会话 ID 等配置，会继续追加对应参数。

也就是说，Claude 这里更像“每个 cc-connect 会话一个长期运行的 CLI 进程”，不是每条消息都 `claude -p` 一次。

### Codex / Cursor：每轮拉起命令，再用 session id 续上下文

Codex 和 Cursor 的模式不同：它们在 `Send()` 时启动一次 CLI 子进程，然后用各自的 resume 机制续上下文。

例如 Codex 首轮类似：

```bash
codex exec <prompt>
```

后续使用：

```bash
codex exec resume <threadID> <prompt>
```

Cursor 则是每次启动：

```bash
agent --print --output-format stream-json --resume <chatID> ...
```

因此，`cc-connect` 抽象上都是“会话”，但底层可以是长驻进程，也可以是每轮启动 CLI 并恢复上下文。

## 第六步：开启 Web 管理端

编译成功后执行：

```bash
./cc-connect web
```

输出：

```text
Web admin is not enabled. Configuring...
Web admin configured on port 9820.
Restart cc-connect for the changes to take effect.
Opening: http://localhost:9820
```

这里容易误解：`web` 命令主要是修改配置并打开浏览器，它并不会让主服务一直在后台运行。因此浏览器访问时出现：

```text
ERR_CONNECTION_REFUSED
```

原因很简单：9820 端口还没有长期监听的服务。

临时前台运行可以执行：

```bash
./cc-connect
```

日志里能看到：

```text
management api started port=9820
cc-connect is running
```

但前台运行会占用终端，关闭或 Ctrl-C 后服务就停止了。

## 第七步：改造成 macOS 后台服务

更合适的方式是使用内置 daemon 命令，把它安装为 macOS `launchd` 后台服务：

```bash
./cc-connect daemon install --config /Users/tom/.cc-connect/config.toml --force
```

安装结果：

```text
cc-connect daemon installed and started.
Platform:  launchd
Binary:    /Users/tom/Desktop/github/cc-connect/cc-connect
WorkDir:   /Users/tom/.cc-connect
Log:       /Users/tom/.cc-connect/logs/cc-connect.log
```

检查状态：

```bash
./cc-connect daemon status
```

结果显示：

```text
Status:    Running
Platform:  launchd
PID:       97867
```

确认 Web 管理端监听：

```bash
lsof -nP -iTCP:9820 -sTCP:LISTEN
```

可以看到 `cc-connect` 正在监听 `*:9820`。再用 HTTP 验证：

```bash
curl -I http://localhost:9820/
```

返回：

```text
HTTP/1.1 200 OK
```

这时刷新浏览器里的 `http://localhost:9820/login?...`，Web 管理端就可以正常打开了。

## 常用后台服务命令

后续日常使用可以记住这几个命令：

```bash
./cc-connect daemon status      # 查看状态
./cc-connect daemon logs -n 100 # 查看最近日志
./cc-connect daemon logs -f     # 跟随日志
./cc-connect daemon restart     # 重启服务
./cc-connect daemon stop        # 停止服务
./cc-connect daemon uninstall   # 卸载后台服务
```

## 小结

这次过程里真正修复了三类问题：

1. **环境问题**：安装 Go，并设置 `GOPROXY` 解决依赖下载。
2. **代码问题**：修复 macOS build tag 冲突，并补齐 launchd 单测夹具。
3. **运行方式问题**：从前台运行切换为 `launchd` 后台服务，让 Web 管理端稳定可访问。

对于类似 `cc-connect` 这种桥接工具，最终能否稳定使用，不只取决于代码能不能编译，还取决于本机 CLI Agent、配置文件、后台服务和端口监听是否串起来。把这些环节逐个验证后，问题就清楚很多了。
