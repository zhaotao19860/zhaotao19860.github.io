---
layout: post
title: "LangGraph 的前世今生、竞争对手和它正在收窄的价值区间"
date: 2026-08-30 21:30:00 +0800
category: AI-Agent
tags: [LangGraph, LangChain, AI Agent, Temporal, MCP]
excerpt: "star 被 Agno 反超，下载量被 Pydantic AI 拉开 1.9 倍，1.0 发布在 HN 只拿到 1 分。但真正在切走它地盘的是上下两层，不是同层对手"
---

先看三个 2026 年 8 月 29 日实测的数字。LangGraph 在 GitHub 上有 40,666 star，PyPI 近 30 天下载 6,790 万次。同一天，Pydantic AI 的 `pydantic-ai-slim` 是 1.281 亿次，差不多是它的 1.9 倍；Agno 的 star 数 41,963，已经反超它。而它 1.0 正式发布那天发到 Hacker News，最终停在 1 分、0 条评论。

这三个数字放一起容易被读成"LangGraph 不行了"。我的判断相反：这是一个东西从"热门框架"变成"基础设施"的典型形状——话题度掉下去，装机量还在，但价值区间正在被上下两层同时切走。

要说清楚这件事，得先回到它为什么会存在。

![cover](/assets/images/ai-agent/langgraph-past-present-future/cover-v1.png)

## 前世：它是来补 LangChain 自己挖的坑

LangGraph 仓库的第一个提交在 2023 年 8 月 9 日，作者是 Nuno Campos，不是 Harrison Chase——Chase 的提交要到六天后才出现。早期内核里根本没有 `StateGraph`，用的是一个叫 `PubSub` 的原语。第一个 PyPI 包 `0.0.8` 发布于 2024 年 1 月 8 日，而且发完就被 yank 了；对外的介绍博文是 2024 年 1 月 17 日。

那篇博文的论点很简单：LCEL 组出来的链是有向无环图，而 agent 的本质是"把 LLM 放进一个 for 循环里跑"，**DAG 表达不了环**。原来的 `AgentExecutor` 把这个循环封起来了，你想改中间任何一步都得钻进抽象。LangChain 内部管这类东西叫状态机。

这个动机在后来几年里被 Chase 本人反复确认，措辞一次比一次直白。2024 年 6 月在 HN 上他说："The initial version of LangChain was pretty high level and absolutely abstracted away too much."2025 年 4 月："We learned this the hard way. This was the issue with the original LangChain chains and agents. They provided abstractions that got in the way."2025 年 10 月，一句更狠的总结："we traded power for ease of use."

**先纠正一个容易读错的地方**。LangGraph 不是"LangChain 的图形化版本"，也不是给 LangChain 加了个可视化。它做的事情正好反过来:把控制流从框架手里交还给你，代价是你要自己写节点、自己声明状态怎么合并。它比 LangChain 更底层，不是更高层。

## 今生的内核：一个回合制的执行模型

Nuno Campos 在 2025 年 9 月写过一篇《Building LangGraph》，讲清了当初排除掉的几个选项：DAG 框架（不支持环）、像 Temporal 这样的持久化执行引擎（他的原话是"as LLM agents get longer and more complicated"，押在那上面是个坏赌注）、一个大 `while` 循环、以及干脆不要算法。

最后选的是 Pregel 那套 BSP 模型：一次 superstep 里，被激活的节点各自拿一份状态副本并行跑，谁都看不见谁的中间结果；回合结束，所有写入按**固定顺序**合并回 channel，再决定下一回合激活谁。固定顺序这件事是刻意的设计目标，Campos 的说法是"output variability should never be the framework's fault"——模型本身已经够不确定了，框架不该再加一层抖动。

![superstep](/assets/images/ai-agent/langgraph-past-present-future/body-superstep-v2.png)

围绕这个内核长出来的几个能力都很扎实。checkpoint 写入在历史长度和线程数上都是 O(1)，所以存档不会随对话变长而变慢；`interrupt()` 做人工介入直接复用了 checkpoint 机制，不需要另一套暂停协议；流式输出有六种模式，能分别拿到 token、节点状态、自定义事件。同样明确的是它不做什么——任务队列被有意留在库外面。那篇文章最后一句我一直记得："The biggest competitor to any code framework is always no framework."

## 版本线：一段真实的 churn 记录

从 0.0.8 到 1.0 走了 21 个月，中间的抖动量比大多数人印象里大：0.0.x 在五个月里发了 69 个版本，0.2.x 发了 76 个，0.3.x 发了 35 个；`0.0.8`、`0.1.18`、`0.3.0`、`0.4.4`、`0.6.9`、`1.1.7`、`1.2.3` 都被 yank 过。几个真正改了写法的节点：

- **0.2.0**（2024-08-07）checkpointer 拆包，`thread_ts` 改名 `checkpoint_id`
- **0.3.1**（2025-02-27）prebuilt 拆包，`ToolExecutor` 换成 `ToolNode`（0.3.0 当天被 yank）
- **0.5.0**（2025-06-26）内部叫 "Getting-Ready-for-1.0"，`state_schema` 变成必填
- **0.6.0**（2025-07-28）Context API 顶掉 `config["configurable"]`，同时引入 durability 三档
- **1.0.0**（2025 年 10 月）执行模型一行没改，唯一的破坏性变更是砍掉 Python 3.9，`create_react_agent` 改叫 `create_agent`
- **1.2.0**（2026-05-12）`DeltaChannel` 进 beta，加上按节点超时和 `error_handler`

顺手说个细节:1.0 的日期有四个版本在流传——PyPI 是 10 月 17 日，npm 是 18 日，文档 changelog 写 20 日，市场博文写 22 日。引用的时候写"2025 年 10 月"最安全。

1.0 最有分量的不是代码，是那句承诺:**2.0 之前不再有破坏性变更**。后面会看到，这在 2026 年是个真差异。另外 Python 和 JS 两条线从 1.0 之后就分叉了，2026 年 8 月底 Python 在 1.2.11，JS 已经跑到 1.4.13，别把两边的版本号当同一件事看。

## 一个多数文章不写的转折：改名和许可

2025 年 10 月那一周，除了 1.0，还发生了另一件事:**LangGraph Platform 改名成 LangSmith Deployment，LangGraph Studio 改名成 LangSmith Studio**，LangGraph Server 改叫 Agent Server。今天去定价页上找，已经没有"LangGraph"这一档了，只有 Developer $0、Plus $39/席/月、Enterprise 按谈，计量单位是 LCU $1.50 和 LSU $1.00，按节点执行次数计费的老方案标成 legacy，老客户的迁移截止日是 2026 年 10 月 1 日。

许可这块的不对称更值得知道。`langgraph` 本身是 MIT，没问题。但真正跑服务的 `langgraph-api` 从它 2024 年 11 月的第一个版本起就是 **Elastic License 2.0**，没有公开仓库（直接 404），以 wheel 形式分发，运行时读 `LANGGRAPH_CLOUD_LICENSE_KEY`；而 `langgraph-runtime-inmem` 也是 ELv2，并且是硬依赖——意味着你在本地敲 `langgraph dev`，跑的就已经是 ELv2 的代码了。有意思的是 JS 那边的 `@langchain/langgraph-api` 反而是 MIT。文档里也有很干脆的话："Double texting is a feature of LangSmith Deployment. It is not available in the LangGraph open source framework."

同一时间线上还有两件事：2025 年 10 月 20 日 LangChain 拿了 IVP 领投的 1.25 亿美元 B 轮，估值 12.5 亿；以及 LangSmith Deployment 现在开始托管**非 LangGraph 的运行时**——Strands、Claude Agent SDK、Google ADK、CrewAI、AutoGen，都能包一层跑上去。

把这几件事连起来读，结论不太舒服但很清楚:母公司自己已经在把赌注从"那个框架"往"那个平台"移。

## 竞争对手：三个方向，而且不在同一层

大部分对比文章只列同层框架，这是最没用的比法。真正在抢 LangGraph 地盘的力量来自三个方向。

### 同层：热闹但都在自伤

2026 年 8 月 29 日实测的横向数：CrewAI star 57,789 最高，Agno 41,963，LangGraph 40,666；下载量上 `pydantic-ai-slim` 1.281 亿反超 LangGraph 的 6,790 万——这个反超的原因我没查到可信解释，`pydantic-ai-slim` 作为被间接依赖的瘦包可能有虚高，别直接当采用度读。

但这一层最关键的事实不是排名，是**14 个月里所有人都发了破坏性大版本**：LangGraph 1.0、CrewAI 1.0、Pydantic AI 2.0、Haystack 3.0、Agno 3.0（后者的升级说明里直接写着 "database migration is required"）。在这个背景下，LangGraph 那句"2.0 前不破坏"就不是营销话术了，它是这一层里唯一敢给的承诺。

另外两个老对手其实已经退出这场比赛了。LlamaIndex 现在自己的仓库简介是 "the leading document agent and OCR platform"，而且它自家的编排是跑在 Temporal 上的；Haystack 转去给欧洲公共部门卖主权 AI。DSPy 经常被拿来对比，但它优化的是提示和权重，跟编排是正交的，不构成替代。smolagents 基本停滞，它主打的 code-as-action 被 Pydantic AI、Agno、Cloudflare、微软当成一个特性直接吸收掉了。

### 上游：厂商不说"不需要框架"，说的是"框架随你，运行时我来"

这是最容易被误读的一层。**微软**：AutoGen 的最后一个版本发布在 Agent Framework 宣布的前一天，MAF GA 四天后 AutoGen 仓库挂上维护模式横幅；但 Foundry 反过来把 LangGraph 列成一等公民托管运行时。**OpenAI**：Agents SDK 到现在还刻意停在 0.22.0 不进 1.0；AgentKit 甚至部分回退了——官方公告写着 "Update on June 3, 2026: OpenAI is winding down the Agent Builder and Evals products"。**Anthropic**：2026 年 4 月 8 日上线 Managed Agents，$0.08/会话小时，论述落在 harness 上，"harnesses encode assumptions about what Claude can't do on its own"。**AWS**：AgentCore GA 后的说法是 "The managed agent harness in AgentCore turns that work into configuration"，外加一句 "Any framework. Any model."

最反直觉的是 **Google**：ADK 2.0 在 2026 年 5 月 19 日**加进了 graph workflows**，文档上还标着 "New in ADK 2.0!"。一个超级云厂商反向收敛到了 LangGraph 的核心抽象上。这说明图这个抽象本身是对的，被验证了；同时也说明它不再是护城河。

值得说明的是，我翻遍了这些厂商的公开表述，**没有任何一家说过第三方框架变得不必要**。他们抢的是运行、部署、计费、可观测这一层，不是编排语义。

### 下层：持久化执行阵营的攻击最有说服力

2026 年 2 月 17 日，Temporal 拿了 a16z 领投的 3 亿美元 D 轮，估值 50 亿，客户名单里有 OpenAI。他们的论点是："Agentic AI doesn't fail because the models aren't good enough… It fails because the systems around them can't handle real-world execution."

Maxim Fateev 那篇《The fallacy of the graph》措辞更硬："A graph is one of the worst ways to represent procedural code","The picture is a lie"。两点必须说清楚:他**从头到尾没点名 LangGraph**，而且文末自己让了一步——简单管道用静态图，动态、数据驱动的 agent 用持久化代码。Restate 的说法更精炼："A library cannot supervise itself. It dies together with the process."

![durability](/assets/images/ai-agent/langgraph-past-present-future/body-durability-v1.png)

Temporal 2026 年 7 月甚至出了官方 LangGraph 插件，介绍里那句话就是全部立论："LangGraph checkpoints state, but checkpoints are not durable execution."

这个攻击最强的证据不来自 Temporal，来自 LangChain 自己的文档和源码：默认的 durability 是 `"async"`，源码注释里写着进程崩溃时"there is a small risk that LangGraph does not write checkpoints"，而恢复是调用方发起的——文档措辞是"你可以从上一个成功的步骤重启你的图"。也就是说存档确实在，但没有任何外部进程守着它、发现它死了、把它拉起来。

反过来 Temporal 也不是没有短板。LangChain 的回应承认持久性上打平，但指出 Temporal 有约 2MB 的 payload 上限；Temporal 自己的工程师也承认 checkpoint 落在 Activity 边界上、不在推理内部，所以一次进行中的模型调用崩了要"从头重试并付两次钱"，并且明确说 "Temporal isn't replacing your framework. It's the layer underneath it."更有意思的是，基础设施厂商往框架层反攻的尝试全都不太顺：Inngest AgentKit 停滞，Hatchet 的 pickaxe 停滞，Temporal 自己的 Agent Harness 还没进预览，31 个 star。

结论是分层而不是替代：编排语义 LangGraph 赢，进程级监督 Temporal 赢，两边都在承认这件事。

## 前景：三条会真正改变结论的线

### 一、能力在往模型和协议里上移

这是最实的一条，而且有量化数据。Anthropic 的 code execution + MCP 把一次工具编排的 token 从 150,000 降到 2,000，降幅 98.7%；Tool Search Tool 砍掉 85% 的工具定义 token，同时把 Opus 4 的 MCP 评测从 49% 提到 74%；Programmatic Tool Calling 再省 37%；Managed Agents 把 p50 首 token 时间降了 60%。微软的 CodeAct 从 27.81 秒 / 6,890 token 降到 13.23 秒 / 2,489 token。Anthropic 自己砍掉了 Claude Code 系统提示词的 80% 以上，LangChain 的 Deep Agents v0.7 也把基础输入 token 降了 65%。

方向很明显：**过去要靠框架小心编排的东西，正在变成模型和协议的内置能力**。MCP 2026 年 7 月 28 日的规范去掉了 `initialize` 和 `Mcp-Session-Id`，改成每请求无状态，加了 `server/discover`；8 月 22 日的路线图里写着 progressive tool discovery——这一项直接吃掉框架在工具管理上的价值。顺便纠正一个过时数字：MCP 注册表现在有 25,590 个 server，Anthropic 官方还在说的 ">10,000" 已经差了 2.5 倍。

A2A 那边 v1.0 在 2026 年 3 月发布，150 多家参与，而且**明确把 LangGraph 列为支持框架**——标准既在削它，也在承认它。

### 二、多智能体明显降温了

这条线的经典对撞只隔了一天：2025 年 6 月 12 日 Cognition 发《Don't Build Multi-Agents》，点名 `swarm` 和 `autogen` 是"错的做法"；6 月 13 日 Anthropic 发文说多智能体在他们的研究系统上比单体 Opus 4 好 90.2%。

一年多之后落点已经清楚。Anthropic 的 Frontier Red Team 在 2026 年 8 月 13 日给出结论："Coordination doesn't naturally emerge from stronger intelligence nor alignment at the individual level."企业侧的规模化比例只有 15%（Deloitte，n=501）和 16%（Menlo，n=495）。最有说服力的是 LangChain 自己也退了:`langgraph-supervisor` 已经软弃用，`langgraph-swarm` 和 `langgraph-bigtool` 处于停更状态。现在还站得住的形态是生成器—验证器分工，不是对等的 agent 群。

### 三、企业侧的数字：先看分母，再看结论

这块流传最广的几个数字都需要还原分母。Gartner 那句"到 2027 年底超过 40% 的 agentic AI 项目会被取消"，底下是一个 3,412 人的**网络研讨会自选投票**；同一份材料里还有一句更值得引用的——"数千家 agentic AI 供应商里只有大约 130 家是真的"。MIT NANDA 的"95% 试点失败"是混合方法的定性研究，它自己的漏斗数据反而推得出约 25% 的试点转生产率。

真正可用的是几个说明了口径的：Menlo 2025 年底的自下而上测算，全年 370 亿美元 LLM 支出里，agent 平台只占 7.5 亿，而 copilot 类是 72 亿，并且"只有 16% 的企业部署和 27% 的初创部署是真 agent"。IDC 2026 年 8 月的数据是另一面：95% 的受访企业至少跑着一个 agent 工作流，平均 11 个，平均月花 117,558 美元，但 **67% 的 agent 预算超支超过 10%，而只有 45.4% 有实时成本看板**。LangChain 自己的调查（n=1,340）里，57.3% 已上生产，最大阻碍是质量占 32%、延迟占 20%，89% 装了可观测但只有 52% 做离线评测、37% 做在线评测。Stack Overflow 2025 的编排工具使用率是 LangChain 32.9%、LangGraph 16.2%。

还有一条和 LangChain 商业模式直接相关的预测：到 2030 年，企业 SaaS 支出中 40% 以上会转向按用量、按 agent、按结果计价。这正是 LCU/LSU 那套计量在赌的东西。

## 我的判断

未来 12 个月 LangGraph 不会消失，理由很具体：需要长任务、需要中途插人、需要可回放审计的团队，在开源侧没有更好的选择，而且它是这一层里唯一给出"2.0 前不破坏"承诺的。

但它的价值区间在明显收窄，三面都在漏：简单管道被厂商原生能力吃掉，复杂长任务被持久化执行从下面接走，集成和工具管理被 MCP 拿走。更关键的信号是母公司自己的重心已经移到 LangSmith Deployment 和 Deep Agents 上了——那个平台连竞品运行时都开始托管。

所以选型判据可以写得很短。**你需要的是确定性的回合制合并、可回放的 checkpoint、和中途能插进人**，那就用它，这三样它做得比谁都干净。**你需要的只是"少写点代码"，或者崩了必须有人自动把它接起来**，那它都不是答案：前者用厂商原生 SDK，后者得在下面垫一层真正的持久化执行。

## 值得盯的几个日期和信号

- **2026 年 10 月 1 日**：LangSmith 老客户定价迁移截止，能看出 LCU/LSU 到底站不站得住
- **2026 年 10 月 31 日 / 11 月 30 日**：OpenAI Evals 转只读、Agent Builder 关停，验证厂商自建编排层的回退幅度
- **2027 年 3 月 31 日**：Foundry 经典 Agents 退役；AutoGen 和 Semantic Kernel 的支持终止下限大约在 2027 年 4 月
- `langgraph-api` **会不会开公开仓库**——这是判断开源承诺边界的单一最强信号
- **默认 durability 会不会从 `"async"` 改掉**——改了说明 LangChain 决定自己吃下持久性，不改说明它接受在 Temporal 那类东西上面跑
- **OTEL 的 `gen_ai.*` 语义约定会不会转 Stable**——现在全部还是 Development 状态、零个稳定属性，这是"直接上 OTEL 就能避免锁定"这个说法目前还不成立的具体原因
- **Temporal 的 LangGraph 插件会不会离开 Public Preview**——离开了，分层格局就算定了
