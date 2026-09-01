---
layout: post
title: "tom-autodev vs LangGraph，领域工作流和编排引擎该怎样分工"
date: 2026-09-01 10:30:00 +0800
categories: [AI-Agent]
tags: [tom-autodev, LangGraph, AI Agent, 研发自动化, iPipe]
excerpt: "tom-autodev 已经有状态机、SQLite、审批和恢复能力。它还要不要迁到 LangGraph，答案取决于两者分别负责什么。"
---

![cover](/assets/images/ai-agent/tom-autodev-vs-langgraph/cover-v1.png)

我最近一直在想一个很具体的问题。`tom-autodev` 已经从一组研发提示词长成了可执行控制面，有状态机、SQLite、审批账本、恢复逻辑和外部系统适配器。LangGraph 也在解决状态、暂停、恢复和长流程。既然两边做的事情开始重叠，是否应该把 `tom-autodev` 改写到 LangGraph 上。

把两边的代码和职责摆在一起以后，我的判断很明确。`tom-autodev` 应该继续保留，LangGraph 可以进入它的执行层，但不该接管它的领域规则。完整重写既浪费已经验证过的工作，也会把关键的研发约束埋进通用图节点里。

这个判断有一个前提。今天的 `tom-autodev` 已经不只是 Skill 文本。当前实现里有 69 个 Python 文件，约 2.8 万行代码，其中包括 27 个测试文件。它用自己的 `Orchestrator` 推进状态，用 SQLite 保存事件、审批、外部操作意图和回执，用 `ArtifactEnvelope` 绑定每个阶段的输入、输出和哈希。LangGraph 在这里面对的已经是一套运行中的控制面，不是一张等待实现的流程草图。

## 两者解决的问题原本就不同

LangGraph 是通用编排库。开发者声明状态、节点和边，运行时按图推进，checkpointer 保存线程状态，`interrupt()` 暂停执行，外部调用再用 `Command` 恢复。它关心的是节点什么时候运行、状态怎样合并、失败后从哪里继续。

`tom-autodev` 关心的是企业研发怎样交付。它从 iCafe 读取需求，经过 Grill、Spec、Tasks、Plan、Implement、Review、iCode、iPipe 和发布。中间设置 G0 到 G10 审批，所有审批绑定输入哈希。Mac 只能生成代码和做源码检查，编译、单测、回归和发布证据只能来自 iPipe。环境失败不能生成业务补丁，Review 意见未经确认不能直接修改代码，连续修复没有进展时必须停下来。

这些规则不会从 LangGraph 里自然长出来。把状态写成 `StateGraph`，并不会自动得到 iCafe 快照、Spec 追踪、双轴 Review、版本绑定、审批失效和远程证据校验。反过来，`tom-autodev` 自己实现这些规则时，也顺手造了一套通用编排设施。两者的重叠点正在这里。

## 放到同一组尺度上比较

| 维度 | tom-autodev | LangGraph |
| --- | --- | --- |
| 核心目标 | 完成受控的企业研发交付 | 执行有状态、可循环的通用工作流 |
| 领域知识 | 内置 iCafe、KU、iCode、iPipe、Review 和项目规则 | 不理解研发平台，需要业务自行实现 |
| 状态推进 | 显式状态表加手写 Orchestrator | `StateGraph`、节点、边和 Pregel 运行时 |
| 持久化 | SQLite 事件、意图、回执、审批和产物分表保存 | checkpointer 保存图状态，store 保存跨线程数据 |
| 人工审批 | G0 到 G10，审批绑定内容哈希和责任人 | `interrupt()` 提供暂停与恢复原语 |
| 外部副作用 | 先写 intent，再写 receipt，并使用幂等键 | 需要节点自行设计幂等和副作用边界 |
| 可观测性 | 领域事件和证据很细，通用运行视图较弱 | 图执行、状态历史和 LangSmith 生态更成熟 |
| 迁移成本 | 已有大量规则和测试，继续扩展会越来越重 | 接入容易，迁移现有控制面仍需重新验证 |

`tom-autodev` 最强的地方是它知道哪些事情不能做。研发自动化最容易出问题的部分很少是模型不会写代码，更多是拿错需求、用旧审批提交了新 diff、把环境故障修成业务代码、重复触发流水线，或者拿另一版代码的成功结果当成本次证据。它把这些问题写进状态、哈希和门禁，通用框架无法替代这部分。

它的代价也已经很明显。每增加一个状态、恢复分支或审批例外，都要同时修改转移策略、Orchestrator、SQLite 表、恢复逻辑、CLI 和测试。当前 `orchestrator.py` 已接近两千行，`state_store.py` 超过八百行。领域规则与运行机制虽然分了模块，维护者仍要同时理解两套东西。继续沿着这条路扩展，控制面的代码量会比研发规则长得更快。

LangGraph 的优势正好落在这块。节点和边让控制流更容易检查，checkpoint 和 interrupt 提供了统一的暂停恢复语义，子图适合隔离 iPipe 等长等待阶段，标准运行时也方便接入跟踪和回放。团队不用继续维护一套自制图执行器。

它也没有消除最难的工作。LangGraph 的 checkpoint 保存图状态，不会替外部 iCode 或 iPipe 操作提供 exactly-once 保证。节点恢复时可能重放，副作用仍要靠 intent、receipt、幂等键和远端查询兜住。它所谓的 durable execution 也需要调用方再次启动图，开源库本身没有外部进程负责发现任务死亡并自动拉起。`sync`、`async` 和 `exit` 三种 durability 只是持久化时机选择，不能替代运行监督。

![tradeoff](/assets/images/ai-agent/tom-autodev-vs-langgraph/body-tradeoff-v1.png)

## tom-autodev 最值得保留的部分

第一部分是 Skill 和子 Skill。Grill 怎样澄清需求，Spec 怎样描述可观察行为，Tasks 怎样切成端到端能力片，Plan 怎样落到仓库、符号、测试和流水线，这些内容需要模型理解语境，也需要项目和语言知识。把它们压成一大组固定状态，只会失去弹性。

第二部分是领域事实。需求快照、ArtifactEnvelope、Change Set、审批记录、iPipe Evidence、Failure Evidence Bundle 都应继续保存在当前领域模型里。它们承担审计和追踪，不能只躺在某个图 checkpoint 的状态字典中。

第三部分是副作用安全。`intent → remote action → receipt` 这套协议要保留。LangGraph 可以决定何时调用节点，不能代替远端系统确认某个提交或流水线是否已经发生。

第四部分是硬边界。Mac 禁止执行项目编译和测试，iPipe 是唯一运行证据，环境故障不生成业务补丁，高风险动作必须经人工审批。这些规则应该继续由 `EvidenceGate` 和适配器边界执行，不能依赖提示词提醒。

## 应该交给 LangGraph 的部分

最适合先迁移的是 `IPIPE → DIAGNOSE → retry or stop` 这一段。它等待时间长，跨会话概率高，重放代价大，分支又相对清楚。可以先做成一个子图，节点只接收领域对象的引用和哈希，不把完整文档塞进图状态。

这个子图大致有五个动作。触发 iPipe，记录远端运行编号，轮询状态，分类失败，等待人工决定是否重跑。图负责路由、暂停和恢复，现有适配器负责查询与副作用，SQLite 领域表继续保存 intent、receipt 和证据。第一阶段跑通以后，再向前接 iCode 提交审批，向后接发布审批。

这里要避开一个很容易出现的坑。现有 `StateStore` 和 LangGraph checkpointer 不能同时声称自己掌握唯一运行状态。更合适的分工是让 LangGraph checkpoint 保存执行游标和短期节点状态，让现有 SQLite 表保存领域事实与审计事件。每次恢复都从领域事实核对图状态，发现哈希或远端证据不一致就停在 `EvidenceGate`，不能猜哪边更新得更晚。

![evolution](/assets/images/ai-agent/tom-autodev-vs-langgraph/body-evolution-v1.png)

## 一条更稳的进化路线

第一步先冻结手写状态机的扩张。新需求若只是增加领域规则，继续放进 Skill、Schema 和 Gate。新需求若涉及等待、循环、并发、暂停或恢复，优先判断能否进入 LangGraph 子图。

第二步给现有控制面加一层 `Engine` 接口。旧的 Orchestrator 作为一个实现，LangGraph 子图作为另一个实现。CLI、如流回调和未来的定时任务只调用 `start`、`tick`、`status`、`resume` 和 `stop`，不直接依赖图框架。这样可以按运行编号灰度迁移，也能随时退回旧路径。

第三步采用一次性 CLI 加定时推进。`submit` 创建运行并返回 `run-id`，`tick` 每隔一两分钟推进所有可运行任务，`status` 和 `list` 提供查询，`resume` 接受人工决定。状态放在数据库里，会话和终端都只是调用者。以当前每天几个到几十个运行的规模，没有必要先引入常驻服务和多租户平台。

第四步只使用 LangGraph 开源库和本地 checkpointer。LangSmith Deployment 带来的托管、计量和许可边界，对当前内部研发流程没有足够收益。观测可以先输出 OpenTelemetry 或现有领域事件，等运行规模真的需要统一平台再做选择。

第五步用可测量结果决定是否继续迁移。第一段子图上线后，至少要看四项指标。相同故障能否稳定恢复，外部操作能否保持零重复，Orchestrator 和恢复代码是否明显减少，旧有证据门禁测试是否全部保留。只换了框架名，代码量和故障面没有下降，这次迁移就没有完成。

最终结构可以很清楚。Skill 定义研发过程和项目知识，领域模型记录事实与证据，LangGraph 推进需要暂停恢复的执行路径，SQLite 保存长期状态，适配器隔离 iCafe、KU、iCode、iPipe 和如流，定时器负责把停住的运行再次唤醒。

## 最后的判断

`tom-autodev` 的优势来自领域约束，缺点来自它承担了太多运行时工作。LangGraph 的优势来自通用执行模型，缺点来自它对企业研发规则一无所知。把前者整体迁成后者，会丢掉已经做完的价值。完全拒绝后者，又会继续支付自研编排引擎的维护成本。

合适的方向是逐段替换运行机制，同时保留 Skill、领域对象、证据协议和安全门禁。第一刀切在 iPipe 的等待与恢复环节。那一段成功以后，LangGraph 才算在 `tom-autodev` 里证明了自己。

## 参考资料

- [LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence)
- [LangGraph 源码中的 durability 定义](https://github.com/langchain-ai/langgraph/blob/main/libs/langgraph/langgraph/types.py)
- [LangGraph 关于同步持久化顺序的公开问题](https://github.com/langchain-ai/langgraph/issues/8039)

![微信公众号](/assets/images/ai-agent/tom-autodev-vs-langgraph/wechat.jpg)
