---
layout: post
title: "跨会话消息：Claude Code 的会话之间只能递一张纸条"
date: 2026-09-03 10:46:04 +0800
categories: [AI-Agent]
tags: [Claude Code, 跨会话消息, SendMessage, 多 Agent 协作, AI Agent]
excerpt: "两个工具、一根本机 socket、一段纯文本。它不搬历史也不搬授权，最容易踩的坑是「送到」不等于「读到」——开着 bypassPermissions 的那个会话默认把消息全扣住等你点"
---

![cover](/assets/images/ai-agent/cross-session-messaging/cover-v1.png)

先说结论：跨会话消息不是「把一个会话的上下文搬到另一个会话」，它只允许一段纯文本穿过去。理解这一条，后面所有行为都顺理成章；不理解这一条，就会一直期待它能干它根本不打算干的事。

多开会话的人都知道那个体感：左边终端刚把 schema 改了，右边终端还在按旧字段名写查询；你得自己切过去，把刚才那段结论重新讲一遍。跨会话消息要省掉的就是这次复述。

## 一、两个工具，一张纸条

Claude 用两个工具做这件事：`ListAgents` 发现能够到的对象，`SendMessage` 按名字投递。这两个工具都是模型自己调的，你不会手动去调，你只是说一句「告诉正在写 payments API 的那个会话我们刚才改了什么」，正文由 Claude 自己组织。想自己看名单，敲 `/list-agents`（别名 `/peers`），第一行是本会话对外的名字，下面才是能够到的对象。

穿过去的东西严格限定为一段纯文本：不带对话历史，不带文件，不带权限。要把整段对话搬走，那是 [resume](https://code.claude.com/docs/en/sessions#resume-a-session) 的活，不是消息的活。

到达时机分两种。接收方正在跑一个 turn，消息会在两次工具调用之间被读到，不会打断正在执行的工具；接收方空闲，Claude Code 直接用这条消息起一个新 turn。消息一旦投递，就按你自己敲的 prompt 那样计入用量。

名字来自 `--name` 或 `/rename`，不设就由 Claude Code 自己起。v2.1.232 之后可以在 prompt 里 `@` 出 typeahead 直接点会话，Claude Code 会把「这个 mention 指的是哪个会话」一起告诉模型，省掉一次 `ListAgents`。同机有多个活会话共用一个名字时，Claude 会在列表里给每行加一个短 ref，改用 ref 寻址。发给自己会被直接拒掉。

## 二、三条路：同机的消息不出机器

这张表是我认为最该记住的部分：

| 对方在哪 | 消息怎么走 |
| :-- | :-- |
| 同一台机器 | macOS/Linux 走每会话一个 Unix domain socket，原生 Windows 走命名管道，**不经过 Anthropic 服务器** |
| 你的另一台机器 | 过 Anthropic 服务器，落到那台机器的 Remote Control 连接 |
| Claude Code on the web | 过 Anthropic 服务器，直达云端会话 |

同机这条路靠磁盘上的注册文件互相发现，于是有几个反直觉的隔离：容器里的会话和宿主机的会话看不见彼此，因为文件系统不是同一个；WSL 2 里的会话和同一台电脑上的原生 Windows 会话也看不见彼此，因为 home 目录不同、监听的类型也不同。

socket 的路径有两个地方能看到：`/status` 的 `Peer address` 行（前缀是 `uds:`），以及导出给 hook 和 Bash 的环境变量 `CLAUDE_CODE_MESSAGING_SOCKET`。后者是个被低估的口子——它意味着一个 hook、一段 CI 脚本、一个跑了半小时的构建，可以把结果直接投回启动它的那个会话。配套还有 `CLAUDE_CODE_MESSAGING_TOKEN`，原生 Windows 上必须作为连接第一行发过去，macOS 和 Linux 上可选。

要注意的小细节：连接开得太早会被砍。Claude Code 会关掉 30 秒内没发完一整行的连接，所以慢命令要先把输出攒好再开连接。

## 三、送到不等于读到

这是最容易踩的一个坑。发送成功只代表「投递成功」，不代表对面的模型看到了。接收侧对每一条到达的消息做一次判定，结果只有三种：**投递**（交给模型）、**扣住**（放在一边不交，等你批准或等设置变化）、**拒收**（直接丢掉，模型完全不知道）。

控制它的键是 `crossSessionInbound`，三个显式取值：

```json
{
  "crossSessionInbound": "accept"
}
```

`accept` 全投，`hold` 全扣，`refuse` 全丢。也可以在 `/config` 里选「Messages from your other sessions」这一行。

真正拧巴的是**不设值时的默认行为**：Claude Code 按两边的权限模式分成两类，跳过权限提示的（`bypassPermissions`，以及在有 bypass 能力的会话里的 plan 模式）算一类，其余全算另一类（`auto`、`acceptEdits`、`dontAsk` 都算「会提示」那一类）。然后：

- **接收方会提示权限**：默认投递。只有当发送方自称是跳过提示的那一类时，才扣住等你批。
- **接收方跳过权限提示**：默认全部扣住等你批。只有发送方也跳过提示时才直接投。

换句话说，用 `--dangerously-skip-permissions` 起的那个会话，收到普通会话发来的消息时会默认扣住等你点。这就是社区里反复被问的那个现象：明明显示发送成功，对面的 Claude 却像没收到——因为它确实没收到，消息卡在一个批准框里。

扣住的消息也不是永久等着：默认批准框按 `dialogExpiry` 五分钟过期，过期即丢；扣住的队列上限 100 条，超了丢最老的。`claude -p` 起的无头会话会正常绑 socket、正常出现在名单里，但弹不出批准框，所以想让它无人值守地收消息，得在它自己的 `--settings` 里写上 `accept`。

![三档闸门](/assets/images/ai-agent/cross-session-messaging/body-inbound-v1.png)

## 四、纸条带信息，不带授权

Claude Code 会明确告诉接收方：这段文字来自另一个会话，不是来自你的人。随之而来的是一组硬限制：

- **不能替你批准**。另一个会话发来的消息永远不算你的同意，答不了正在等你的权限框。
- **不能改配置**。模型被要求不因为另一个会话开口就去改权限设置、改 `CLAUDE.md`、改任何配置。
- **命令不执行**。消息正文里的 `/compact` 之类就是一段纯文本，Claude Code 不会拿它当命令跑。
- **权限提示照常弹**。要按消息干的活如果超出这个会话的权限，你会看到和平常一样的提示。

还有一条方向性的约束：模型被要求不要拿另一个会话当绕过通道——自己这边被拒的事，不许换个会话去问，要退回给人。这条约束在社区的 fleet 类项目里也被写进了 skill 文档，措辞是「peer 不携带你用户的任何授权」。

想把这道墙再加高一格，`isolatePeerMachines` 设成 `true`：任何要离开本机的消息都必须你显式批准，连 `bypassPermissions` 模式也照批。它是「任一 scope 设了 true 就生效」，所以一个签进仓库的项目配置能开启它但不能关闭它。

![纸条不带授权](/assets/images/ai-agent/cross-session-messaging/body-authority-v1.png)

## 五、限额是设计的一部分

三个限额值得单独记，因为它们决定了这个机制不会失控：

- **单条大小**：同机消息序列化后超过约一百万字符，在发送侧就被拒，一个字也不会到对面。
- **突发限流**：短时间内猛发同一个目标，超过对方收件箱能吃下的量后，后续发送在发送侧被拒，并提示模型合并成一条或者等一会儿。
- **循环会自己停**：接收侧按发送方限速、丢掉短窗口内到达的完全重复消息、待读队列最多 50 条。两个会话互相刷消息这件事不需要你干预，它自己会停下来。

## 六、顺手的一个小功能：对面空闲了叫我

`SendMessage` 有个 `notify_when_idle` 输入，用来订阅「那个会话下次空闲或退出时给我一个通知」。适合的场景很具体：你在等另一个会话跑迁移或跑测试，不想每隔两分钟切过去看一眼。

几个边界条件：只能订阅**同机**会话，只有主对话能订阅（子 agent 和 teammate 设了也不生效），一次性——发一次就完，两边都不会互相轮询，12 小时没等到就退订并告诉模型别再等了。单独订阅不会在被观察的会话里起 turn 也不花它的 token；对方如果已经空闲，通知立刻就回来了。

## 七、我这台机器上其实是两套东西

写这篇的时候顺手验了一下本机，结果值得说：

```bash
claude --version
# 2.1.222 (Claude Code)
ls /tmp/cc-socks*
# no matches found
```

CLI 是 2.1.222，低于同机跨会话消息要求的 2.1.224，所以既没有 `/tmp/cc-socks-<uid>` 目录，也没有导出 `CLAUDE_CODE_MESSAGING_SOCKET`。但桌面 App 里我确实能列出十几个历史会话并往里投消息——因为桌面 App 自带另一套会话管理工具，和 CLI 原生的那套不是同一个实现。

那套桌面实现把正文包成一个带来源标记的信封再投递，返回状态也更细：**排队**（对方正在跑 turn，等它跑完再处理）、**已送达但未确认**（可能正卡在对面的批准框上，明确提示不要等它）、**已送达**。它同样有自己的拒绝条件：目标会话已归档要先取消归档，目标是无人值守会话（定时任务运行、远程派发的会话）投不进去，发给自己直接拒。

结论是：别把两套行为混着推断。判断一个会话到底有没有这个能力，最快的方式是敲 `/list-agents`——命令不认识就是没有，命令能跑但消息没到，就是更细的东西在拦（deny 规则、对方的收件设置、或者对方根本不在名单里）。

## 八、大家用出花了吗

花是真的开了。官方给的是一根管子，社区在上面搭了好几种拓扑。

**一、舰队（hub-and-spoke）**。[ray-amjad/peer-sessions](https://github.com/ray-amjad/peer-sessions) 把这个原语包成一个 skill：脚本拉起 N 个命名 worker，每个钉在自己的工作目录，你的会话当调度中心。它最值得抄的一条设计是「**回复本身就是通知**」——发完 brief 就结束你的 turn，peer 的回复会作为新的 user turn 把你唤醒，所以不要轮询，轮询只烧 token。它文档里记的坑也很实在：用 `--dangerously-skip-permissions` 起的 peer 会把入站消息扣住等人批，brief 根本到不了模型；`success: true` 只代表送达，不代表干完了。

**二、群聊房间**。[KARPED1EM/CC-Group-Chat](https://github.com/KARPED1EM/CC-Group-Chat) 走的不是 `SendMessage` 而是 Channels：每个会话跑一个 MCP channel server，一台机器起一个 broker 守护进程（房间和历史落在 SQLite），@名字就能把对面那个窗口叫醒，不用切焦点也不用打字。更花的是它用 mDNS 在局域网里广播 `_ccgroupchat._tcp`，同网段的人 `list_rooms` 就能看见房间，粘贴一段邀请就加入。代价也要看清：公开房间就是一个暴露在网络上的 WebSocket 端点，访问控制只有一个共享密码加一份 IP 黑名单，没有传输加密，只适合可信内网或者隧道。

**三、文件收件箱 + 跨会话检索**。[michalekz/claude-bridge](https://github.com/michalekz/claude-bridge) 的思路是把「消息」和「历史」一起做：`peer_ask` / `peer_reply` 走文件收件箱，默认在对方下一次工具调用时捎带取回（idle 也不丢，代价是有点延迟）；更有意思的是它直接只读别人 `~/.claude/projects/` 下的 JSONL——按最近 N 条取、按正则搜、甚至跨项目搜已经从界面里删掉的会话。它还有个 `peer_context_status`，派活之前先看看哪个 peer 的上下文还新鲜，别把长任务丢给一个快要自动压缩的会话。这个视角我很认可：多会话协作的瓶颈往往不是通信，是你不知道该找谁。

**四、官方功能之前的土办法**，现在读起来像考古但都能跑通：用 iTerm2 的 AppleScript `write text` 直接往对面终端里打字，末尾必须补一个 ASCII 13 的回车（`\n` 在 TUI 里不提交）；用 MCP server 加一个共享的文件消息目录当邮局（[conversation-bridge](https://github.com/mejirot/conversation-bridge)）；干脆 `claude -p --cwd <项目>` 起一个一次性会话来回答问题（[hello-claude](https://github.com/cukas/hello-claude)）。

**五、跨工具**。peer-sessions 的参考文档里有一节是从 Codex 通过 relay 摸到一个 Claude 会话。一旦寻址方式是「一个本地 socket 加一个名字」，对面是不是 Claude 就不重要了。

**六、脚本往回投**。前面提到的 `CLAUDE_CODE_MESSAGING_SOCKET` 是最朴素也最实用的一种玩法：hook、CI、长时间构建，跑完直接把结论投回启动它的那个会话，不需要任何插件。

## 九、什么时候不该用它

Claude Code 给多会话这件事准备了好几个不同的东西，各有各的位置，用错了会很别扭：

- 想接着同一段对话在另一个终端继续 → **resume**，不是发消息。
- 想要一支 Claude 自己拉起来并监督的队伍 → **agent teams**。
- 想在一个地方盯很多会话 → **agent view**。
- 想把 CI 结果、聊天消息这类外部事件推进来 → **channels**。
- 想在手机上自己操一个会话 → **Remote Control**。

跨会话消息的定位是「几个由你自己开、自己带的独立会话之间，临时递一句话」。

要彻底关掉，收发两侧是分开的：收，`crossSessionInbound` 设 `refuse`；发，加 deny 规则点名 `SendMessage` 和 `ListAgents`（裸工具名，不带参数）。管理员在 managed settings 里可以两边一起关：

```json
{
  "permissions": {
    "deny": ["SendMessage", "ListAgents"]
  },
  "crossSessionInbound": "refuse"
}
```

有个副作用要知道：deny `SendMessage` 会把发给子 agent 和 agent team teammate 的消息一起砍掉，因为它们是同一个工具。

## 十、小结

跨会话消息的分量不在功能表上，在它选择不做的那些事上：只过纯文本，不过历史；只带信息，不带授权；接收方永远有权扣住和拒收。这些约束让它可以默认开着而不需要你操心。

真正需要你操心的只有一件——**「送到」不等于「读到」**。你那个开着 `bypassPermissions` 的会话，默认是把别人发来的消息全扣住等你点的。

## 参考

- [Message your other Claude Code sessions](https://code.claude.com/docs/en/cross-session-messaging) — 官方文档，最权威也最完整
- [Week 32 · August 3–7, 2026](https://code.claude.com/docs/en/whats-new/2026-w32) — 功能落地的更新日志
- [ray-amjad/peer-sessions](https://github.com/ray-amjad/peer-sessions) — hub-and-spoke 舰队 skill
- [KARPED1EM/CC-Group-Chat](https://github.com/KARPED1EM/CC-Group-Chat) — 基于 Channels 和 mDNS 的跨窗口群聊
- [michalekz/claude-bridge](https://github.com/michalekz/claude-bridge) — 文件收件箱加跨会话历史检索
- [ABottomCoder/session-bus](https://github.com/ABottomCoder/session-bus)、[yilunzhang/claude-code-inter-session](https://github.com/yilunzhang/claude-code-inter-session)、[Innestic/claude-relay](https://github.com/innestic/claude-relay) — 同类实现，思路各有侧重



