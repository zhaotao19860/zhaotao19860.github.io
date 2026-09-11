---
layout: post
title: "Tom-Autodev、AIFlow、LangGraph 与 Langfuse，四种 AI 工程工具怎么分工"
date: 2026-09-11 18:00:00 +0800
categories: [AI Agent]
tags: [Tom-Autodev, AIFlow, LangGraph, Langfuse, Agent, 研发自动化]
---

![Tom-Autodev、AIFlow、LangGraph 与 Langfuse 的分工](/assets/images/ai-agent/tom-autodev-aiflow-langgraph-langfuse/cover-v1.png)

最近把 Tom-Autodev、AIFlow、LangGraph 和 Langfuse 放在一起比较，最容易掉进一个坑。它们都能和 Agent、工作流、节点、日志这些词扯上关系，于是看起来像四个同类产品。

其实它们回答的是四个不同的问题。

Tom-Autodev 关心一次研发需求怎样从入口走到交付。AIFlow 关心一套 AI 协同研发流程怎样被编排、复用和监控。LangGraph 关心开发者怎样构造一个有状态、可暂停、可恢复的 Agent 应用。Langfuse 关心这个应用运行以后究竟做了什么，效果和成本怎么样。

```text
Tom-Autodev  研发交付体系
AIFlow       AI 研发工作流平台
LangGraph    Agent 应用编排框架
Langfuse     LLM 应用可观测性与评测平台
```

这四者可以组合，也可以分别使用。真正的选型问题不是谁更强，而是哪一层的问题正在阻塞你。

## Tom-Autodev 负责把需求送到交付

Tom-Autodev 的起点是一张研发需求，终点是经过验证的代码交付。它把需求澄清、Spec、任务拆解、业务代码实现、产品测试、代码审查、iCode 提交和 iPipe 验证组织成一条有状态的研发路径。

这里的关键对象是需求、任务、评审意见、测试证据和发布状态。每一步都要回答一个工程问题。需求是否已经说清楚，方案是否获得确认，代码是否真的改到了正确的仓库，测试是否覆盖了需求，Review 的问题有没有关闭，提交是否关联了正确的卡片。

这使 Tom-Autodev 更像一套研发交付操作系统。它的价值来自流程约束和证据链，Agent 只是其中负责理解、生成、检查和修复的执行者。

它适合这些场景。

* 团队已经有明确的研发制度，需要让 Agent 遵守已有流程。
* 一次需求涉及多个仓库、业务代码和独立测试库。
* 交付需要经过人工确认、代码审查、流水线和发布门禁。
* 失败以后要知道回到哪一步，不能只重新开一个聊天窗口。

Tom-Autodev 的边界也很明确。它不是一个通用的 Agent 图编程框架，也不是面向所有 LLM 请求的追踪系统。你若只是要做一个带工具调用的客服 Agent，用它会显得过重。

## AIFlow 负责把研发流程编排成可运行的图

AIFlow 把研发流程写成 YAML 工作流。流程中可以同时放 AI 节点、确定性节点和人工审批节点。

AI 节点负责需求分析、规划、代码生成和审查。确定性节点负责脚本、测试、Git 操作等不该交给模型自由发挥的事情。审批节点在关键决策处暂停，等待人确认后继续。

它还有几个很适合研发任务的设计。

第一是节点级上下文隔离。每个节点运行在独立的会话中，减少一个长会话越聊越乱的问题。第二是产物目录。上游节点把分析报告、方案和测试结果写到固定位置，下游节点显式读取。第三是结构化输出。节点产出的状态可以参与后续的 `when` 判断。第四是 `loop_back`。测试失败时，流程可以回到修复节点，再执行测试，直到成功或达到次数上限。

![AIFlow 把 AI 节点、确定性节点和审批节点串成研发闭环](/assets/images/ai-agent/tom-autodev-aiflow-langgraph-langfuse/body-aiflow-v1.png)

AIFlow 和 Tom-Autodev 的关系，比较像平台和方法的关系。Tom-Autodev 定义研发交付要经过哪些门，AIFlow 提供一套把多个能力单元串起来运行的机制。一个 Tom-Autodev 流程可以借助 AIFlow 编排，AIFlow 也可以编排代码评审、测试生成等不属于 Tom-Autodev 的流程。

## LangGraph 负责构造 Agent 的运行逻辑

LangGraph 面向的是开发者。它把 Agent 应用表示为状态图，节点读取当前状态并返回状态更新，边决定下一步走向。图可以有条件分支，也可以有循环。

这套模型适合需要记住中间状态的 Agent。比如一个研究 Agent 先搜索资料，再判断资料是否足够，资料不足就继续搜索，资料足够后生成答案，最后等待人工确认。通过 checkpointer 保存状态后，应用可以在中断、失败或进程重启后恢复执行。`interrupt` 则让开发者在图中安排人工介入。

LangGraph 解决的是应用内部的控制流。它让开发者明确写出 Agent 怎样思考、调用工具、更新状态和处理分支。代价是开发者需要自己设计状态模型、节点边界、幂等性、权限和部署方式。

所以 LangGraph 更适合下面的任务。

* 需要自定义 Agent 行为，而现成工作流模板不够用。
* 需要循环、并行、多个 Agent 协作或复杂的状态转换。
* 需要把 Agent 作为产品能力嵌入自己的服务。
* 团队愿意承担运行时、存储和部署方面的工程工作。

AIFlow 的 YAML 更接近研发人员可以直接维护的流程描述。LangGraph 的代码模型更接近软件工程师构造一个可编程运行时。两者都能表示 DAG，也都能做循环和人工介入，但使用者和交付边界不同。

## Langfuse 负责回答运行之后发生了什么

Langfuse 位于另一层。它通过 tracing 记录一次请求中的模型调用、工具调用、检索、子步骤和最终输出，并把这些步骤串成可以查看的运行轨迹。

对于 Agent，单看最终回答通常不够。一次失败可能来自错误的工具选择、检索结果为空、提示词版本变化、上下文过长、模型重试过多，或者某个分支陷入循环。Langfuse 的追踪和 Agent 图视图，可以帮助开发者还原这次运行经过了哪些节点。

它还提供 Prompt 管理、版本关联、数据集、评分和评测能力。这样可以把“这次回答感觉变差了”变成可追踪的问题。你可以知道某个版本的 Prompt 产生了哪些结果，也可以比较不同模型或不同 Prompt 在同一批数据上的表现。

![LangGraph 运行 Agent，Langfuse 记录轨迹与质量反馈](/assets/images/ai-agent/tom-autodev-aiflow-langgraph-langfuse/body-layered-stack-v1.png)

Langfuse 不负责替你决定业务流程，也不负责把 iCafe、代码仓库、测试库和发布流水线串起来。它记录和分析运行事实，帮助团队发现问题、评估改动和控制成本。

## 放在一起看，组合关系更清楚

假设团队要做一个“需求到代码提交”的 AI 研发流程，可以这样分工。

Tom-Autodev 规定交付阶段、审批要求和研发证据。AIFlow 把这些阶段编排成工作流，负责节点执行、产物传递、暂停、重试和失败回环。某个节点内部如果需要复杂的多 Agent 推理，可以用 LangGraph 实现。运行过程中的模型调用、工具调用、耗时、Token 和评测结果，再接入 Langfuse。

```text
需求与交付规则
        ↓
Tom-Autodev
        ↓
AIFlow 工作流
        ↓
LangGraph Agent 节点
        ↓
Langfuse 追踪、评测与成本分析
```

这不是要求四个工具必须一起上。小团队完全可以先用 LangGraph 加 Langfuse 做出一个 Agent 应用。研发流程已经高度标准化的团队，可以优先建设 Tom-Autodev 和 AIFlow。真正需要统一 Prompt 版本、排查 Agent 轨迹和做离线评测时，再补上 Langfuse。

## 怎么选

如果你的问题是“一个需求如何合规地完成并交付”，优先看 Tom-Autodev。

如果你的问题是“怎样把多个 AI 能力和脚本编排成可复用的研发流程”，优先看 AIFlow。

如果你的问题是“怎样在代码里实现一个有状态、可恢复、支持人工介入的 Agent”，优先看 LangGraph。

如果你的问题是“模型为什么这样做，质量是否下降，成本从哪里涨起来”，优先看 Langfuse。

我更愿意把它们看成一条逐渐变长的工程链。前两者把研发活动组织起来，LangGraph 把 Agent 的行为写清楚，Langfuse 把运行结果留下来供人分析。工具越多，系统能力越强，集成和治理成本也会一起上升。选型时先找到当前最缺的那一层，通常比一次性搭完整套件更稳。

### 参考资料

* [AIFlow 平台分享](https://ku.baidu-int.com/knowledge/HFVrC7hq1Q/OhmTcK45Eq/amAslPrRVg/KlKCIqrblNNQjH)
* [AIFlow 详细使用手册](https://ku.baidu-int.com/knowledge/HFVrC7hq1Q/OhmTcK45Eq/amAslPrRVg/mQ8ga3JnMNa-y5)
* [AIFlow 实践](https://ku.baidu-int.com/knowledge/HFVrC7hq1Q/9IaD11zSiz/Xa05s1Ibuy/F6pw07S8HpcjTh)
* [Langfuse Agent Observability](https://langfuse.com/blog/2024-07-ai-agent-observability-with-langfuse)
* [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
