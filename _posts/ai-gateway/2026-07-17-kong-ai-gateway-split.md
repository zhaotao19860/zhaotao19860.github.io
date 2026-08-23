---
layout: post
title: "AI 网关开始独立成产品，Kong 为什么要和 API 网关分家"
date: 2026-07-17 22:23:23 +0800
category: AI-Gateway
tags: [AI Gateway, Kong, API Gateway]
excerpt: "独立的不只是版本号：模型、MCP、Agent 正在把入口层拆成三种治理语义"
---
![](/assets/images/ai-gateway/kong-ai-gateway-split/01.png)

过去三年，AI Gateway 常见的落地方式，是在 API Gateway 上再挂几组插件：识别 token、切换模型、过滤 prompt、做语义缓存。入口还是那个入口，只是多懂了一点 AI。

Kong AI Gateway 2.0 改了这个前提。它不再沿用 Kong Gateway 3.x 的版本线，而是拥有自己的 runtime、control plane、Admin API、版本号和发布节奏。更值得注意的是，管理对象也从 Service、Route 转向 Model、MCP Server、Agent。

这不是一次普通升级，而是入口层开始按流量语义重新分家。

## Kong 拆开的其实是两只时钟

Kong 在[官方公告](https://konghq.com/blog/product-releases/kong-ai-gateway-2-0-agentic-ai)里把矛盾说得很直接：传统 API Gateway 承担关键业务基础设施，适合稳定优先的季度节奏；模型供应商和 Agent 协议则以周为单位变化。让两类工作负载共用一个代码发布列车，一边会嫌慢，另一边会嫌冒险。

所以 2.0 的变化有三层。

产品层，AI Gateway 从 Kong Gateway 里的能力集合变成独立产品线。运行层，它有专门的 data plane、control plane 和 analytics pipeline。管理层，新 Admin API 直接围绕 Models、MCP Servers、Agents，以及 Policy、Consumer、Provider、Vault 等对象工作，不再要求团队先把 AI 概念翻译成 Service 和 Route。

这套 2.0 目前还是 private beta，官方计划 7 月底 GA。既有 AI Gateway plugins 不会立刻消失：Kong Gateway 3.14 是 LTS，旧配置仍可运行；到 3.18，AI plugins 才会从默认捆绑改为 opt-in。`kongctl` 则负责把现有 decK 配置转换为 2.0 原生配置。

因此，“分家”不是今天拔掉旧网关，而是先把新产品的演进权、对象模型和故障边界划出来。

## 为什么 Model、MCP、Agent 不能再混着管

三类流量都可能经过 HTTP，但 HTTP 外壳相同，不等于治理语义相同。

**Model 流量的核心对象是一次推理。** 平台关心 provider、model、输入输出 token、首 token 延迟、总延迟、流式中断、成本、内容安全，以及重试会不会造成重复计费。它的路由目标通常是“把这次生成交给谁”。

**MCP 流量的核心对象是工具与会话。** MCP 规范把它定义为基于 JSON-RPC 的有状态 client-host-server 协议：client 与 server 建立 session，初始化时协商 capability，运行中既有 `tools/call`，也可能有 sampling、通知和 server-to-client 请求。远程 Streamable HTTP 还涉及 SSE、重连、`MCP-Session-Id` 和消息重投。它的授权问题不是“能否访问这个 URL”，而是“这个身份能否看到、调用这个 tool，参数和返回数据是否越权”。

**Agent 流量的核心对象是任务与委托。** 以 A2A 为例，协议包含 Agent Card、Task 状态、Message、Artifact、流式更新和 push notification。一次调用可能在 `working`、`input-required`、`auth-required` 与终态之间迁移，还要保留用户身份、委托链和任务上下文。它的治理问题是“谁代表谁做了什么，任务现在停在哪里”。

把这三套语义落到策略上，差异会更具体。

Model 入口的策略键通常是租户、模型别名、token 预算、数据分类和 provider 健康度。故障处理关注限额、超时、fallback 与幂等边界：同一请求切到备用模型，结果可能变化，账单也可能增加，所以“自动重试”本身就是需要审计的决策。

MCP 入口的策略键则是 server identity、tool name、OAuth scope、capability 与 session。`tools/list` 决定 Agent 能发现什么，`tools/call` 决定它能执行什么，两者必须使用一致的权限视图；否则会出现“列表里看得见，调用时才被拒绝”，或更危险的“列表里隐藏，仍可按名称直调”。网关还要区分连接中断、session 过期与显式取消，不能看到 TCP 断开就假设业务已经终止。

Agent 入口需要再加一层 task、context、delegation chain 和 human approval。策略不只在请求进入时判断，还可能在任务运行中再次判断：Agent 先获得检索权限，生成购买方案后，真正提交订单前应重新验证用户身份、金额范围与确认记录。若只在第一跳鉴权，后续 Agent-to-Agent 委托很容易把原始权限扩大成一张长期通行证。

![](/assets/images/ai-gateway/kong-ai-gateway-split/02.png)

如果把三者压进同一套“请求数、状态码、Route、Consumer”模型，指标会失真。Model 的 200 可能只代表流开始；MCP 的连接断开不等于调用取消；Agent 的 HTTP 响应结束，也不代表任务完成。统一入口仍然有价值，但不能只剩统一字段。

## 独立产品真正释放了三个信号

第一个信号是，**对象模型比插件数量更重要**。过去说 AI Gateway，容易列出限流、缓存、guardrail、model routing 等功能。2.0 更关键的变化是控制面承认 Model、MCP Server、Agent 是不同实体。实体一旦不同，生命周期、策略绑定、审计索引和所有者也会不同。

第二个信号是，**发布节奏本身成为架构约束**。协议适配器、provider connector 和成本策略需要快；证书、基础路由与关键 API 变更需要稳。把所有能力锁在同一版本线上，会让快层绕过平台，或让稳层被迫承受高频升级。

第三个信号是，**组织边界会跟着控制面变化**。API 平台团队过去管理 Service、Route 和 Consumer；AI 平台团队还要管理模型目录、token 预算、tool registry、Agent 身份与协议兼容。独立 control plane 让两支团队可以共享身份、策略和运维方法，却不必共享每一次发布。

这也改变了故障归属。模型提供商抖动、MCP server 返回错误 schema、Agent 长时间停在 `input-required`，表面都可能表现为“AI 请求慢”，处理人却分别属于模型平台、工具所有者和业务 Agent 团队。控制面若能把流量挂到原生实体，告警才有机会直接路由给正确责任人，而不是全部落到 API 网关值班群。

![](/assets/images/ai-gateway/kong-ai-gateway-split/03.png)

不过要克制一个推论：Kong 把产品拆开，不等于每家公司都应该立刻部署三套物理网关。真正需要拆的是治理域；runtime 是否分开，要看故障隔离、升级频率、合规边界和团队规模。

## 我的技术判断：先语义分治，再决定物理解耦

**Kong AI Gateway 2.0 最重要的信号，不是“API Gateway 过时了”，而是 API 治理模型已经不足以完整描述 AI 入口。** API Gateway 仍然适合稳定的请求路由、身份入口与东西向策略；AI Gateway 则要在其上建立对推理、工具会话和任务委托的原生理解。

平台团队可以先做四件事。

第一，把流量盘点成 Model、MCP、Agent 三类，分别定义主对象和终态：推理完成、tool session 结束、task 到达终态，不能都用 HTTP 200 代替。

第二，拆策略。Model 按 token、成本和内容风险治理；MCP 按 server、tool、scope、session 治理；Agent 按身份委托、Task 状态、Artifact 去向和人工确认点治理。

第三，拆 SLO 与升级窗。provider 适配可以周更，核心 API 路由保持季度稳定；MCP/A2A 的协议兼容变更必须有 canary 和回滚单元。

第四，再评估部署边界。如果 AI 流量已经影响 API 网关升级，或两者的合规、容量和故障半径明显不同，就值得独立 runtime；如果规模尚小，先在同一平台内做配置、指标和所有权隔离，成本更低。

迁移时也不要先追求功能对照表。传统 Service/Route 配置能被工具转换，不代表旧日志、告警、容量模型和应急手册会自动继承。至少要并行核对四组结果：同一身份是否命中同一政策，同一模型请求是否记录相同 token，同一 MCP session 是否能在断线后恢复，同一 Agent task 是否能从入口追到最终 Artifact。只有这些运行语义对齐，才算完成迁移，而不是“配置成功导入”。

判断是否真的需要“分家”，可以问一句：一次 Agent 任务失败时，你能否沿着用户身份找到模型调用、MCP tool、Task 状态和最终 Artifact，同时不让 AI 侧的高频变更拖着核心 API 一起升级？

如果答案是否定的，问题已经不再是少装了一个插件，而是入口层的抽象还停在上一代。

## 参考来源

- [Kong AI Gateway 2.0 官方公告](https://konghq.com/blog/product-releases/kong-ai-gateway-2-0-agentic-ai)

- [Kong AI Gateway 3.14 官方发布](https://konghq.com/blog/product-releases/kong-ai-gateway-3-14)

- [Model Context Protocol Architecture](https://modelcontextprotocol.io/specification/2025-11-25/architecture)

- [MCP Streamable HTTP Transport](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)

- [A2A Protocol v1.0 Specification](https://a2a-protocol.org/v1.0.0/specification/)

- [Envoy v1.39.0 Release](https://github.com/envoyproxy/envoy/releases/tag/v1.39.0)
