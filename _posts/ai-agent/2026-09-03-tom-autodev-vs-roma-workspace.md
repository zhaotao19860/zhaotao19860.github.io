---
layout: post
title: "tom-autodev 与 Roma Workspace，谁在管什么"
date: 2026-09-03 18:00:00 +0800
categories: [AI-Agent]
tags: [Comate, Agent, Roma Workspace, tom-autodev, 研发自动化]
---

![tom-autodev 与 Roma Workspace](/assets/images/ai-agent/tom-autodev-vs-roma-workspace/cover-v1.png)

我第一次把 tom-autodev 和 Roma Workspace 放在一起看时，也很容易把它们当成两套研发工作流。两边都有需求澄清、规格、任务拆解、实现、验证，也都提到了 Skills、Subagents 和 Worktree。若只看名字和流程图，它们像在争同一份工作。

把实际文档和代码对起来以后，边界清楚了。Roma Workspace 负责组织团队交给 Agent 的研发资产，tom-autodev 负责控制一个具体需求如何通过审批、生成证据并走到发布。前者回答 Agent 来到这个项目以后能看到什么、应该调用什么，后者回答这次交付进行到了哪里、凭什么继续、失败后回到哪一步。

## Roma Workspace 管团队怎样教会 Agent

Roma Workspace 的核心是一座围绕 Agent 建设的团队研发空间。它可以是单仓，也可以用一个指挥仓关联多个业务仓。业务代码仍在原来的仓库中，指挥仓集中保存 Agent 做事时缺少的说明和工具。

`docs/` 保存产品设计、视觉约束和环境信息。`repos/` 给每个代码库准备 `overview.md`、`setup.md`、`test.md`，让 Agent 知道仓库职责、构建方式和测试办法。`skills/` 收纳需求讨论、规格转换、实现、排障和评审等可复用流程。`hooks/` 提供硬约束，比如改了 B 目录必须检查 A 目录，或者禁止在旧框架目录里新增代码。`AGENTS.md` 只描述 Agent 在这个空间里怎样工作，以及何时使用 Subagent。

![Roma Workspace 的资产组织](/assets/images/ai-agent/tom-autodev-vs-roma-workspace/body-roma-assets-v2.png)

这套设计最有价值的地方，是把团队成员脑中的经验变成 Agent 能读、能调用、能检查的资产。产品、研发和测试也不必只维护各自那一段交接材料。大家共同维护的是一套能够支撑完整研发活动的上下文。

Roma 的参考流程也覆盖了开发全过程。需求清楚、范围小、风险低时可以直接进入实现。需求还需要讨论时，先用 `roma-grill-with-docs` 或 `roma-wayfinder` 梳理决策，再转成 Spec。任务太大时拆成 Tickets，遇到怪问题时进入结构化诊断，会话装不下时用 handoff 交接。需要隔离代码时再借助 Worktree 或云端沙箱。

但 Roma Workspace 仍然是一套建设方法。文档本身也提醒了它的代价。Docs、Skills 和 Subagents 堆得太多，过期知识可能比 Agent 能力更新得更快。跨仓库 Diff 目前也不够直观，明确要改某一个仓库时，缺少一步直达的体验。团队要留下真正有用的东西，也要控制数量。

## tom-autodev 管一次交付能否继续

tom-autodev 的关注点更窄，也更硬。它从一张经过人工确认的 iCafe 卡片开始，沿着固定状态机推进。

```text
Intake
  -> Grill
  -> Spec
  -> Tasks
  -> WorkspaceGate
  -> Plan
  -> Implement
  -> Review
  -> iCode
  -> iPipe
  -> Release
```

每一步都会生成结构化产物。Grill 输出决策记录和补充验收条件，Spec 输出行为、测试接口、环境要求和追踪关系，Tasks 输出无环任务图，Plan 精确到仓库、文件、符号和测试用例，Implement 同时产出业务补丁与独立测试补丁，Review 分别检查 Standards 和 Spec，Diagnose 则要求给出可证伪的单一根因。

这些产物放进带版本和内容哈希的 `ArtifactEnvelope`。后一步发现输入哈希、基线版本或环境指纹对不上，就拒绝继续。阶段文档写入知识库，链接和哈希回填到 iCafe 评论，外部动作在执行前写 intent，执行后写 receipt。系统恢复时只认已经确认的结果，不能从一个悬空动作猜测成功。

![tom-autodev 的交付闸门](/assets/images/ai-agent/tom-autodev-vs-roma-workspace/body-autodev-gates-v1.png)

人工审批同样是状态机的一部分。G0 确认需求卡片和项目绑定，G1 处理需求决策，G2 审 Spec 和测试接口，G3 审任务图，G4 审每个任务计划，G5 审完整 Diff，G6 审修复方案，G7 批准 iCode 提交，G8 批准失败阶段重跑，G9 批准发布。每次审批都绑定具体输入哈希，内容变化后旧审批随即失效。

它对执行证据的要求也很明确。Mac 只用于读代码、生成补丁、源码评审、管理 Worktree 和解析流水线证据。编译、单测、回归、集成验证和发布都必须进入配置好的 iPipe 环境。Review 只提供证据，无权代替人批准提交。代码失败和测试失败先进入诊断，环境失败不能顺手改业务代码。

这些约束会让流程显得重。它解决的正是多人、多仓、长周期交付中最难追问的几件事。现在审的是哪一版，测试跑的是不是同一组 Revision，失败来自环境还是代码，恢复以后会不会重复提交，谁批准了哪一次发布。

## 两者的关系更像空间与控制器

Roma Workspace 的覆盖面较宽。它可以容纳很多研发方式，也允许团队逐步形成自己的 Docs、Skills、Hooks 和 Agent 使用习惯。tom-autodev 的路径较固定，它把 iCafe、知识库、iCode、iPipe 和如流审批串成一条可恢复、可审计的交付链。

| 维度 | Roma Workspace | tom-autodev |
| --- | --- | --- |
| 主要对象 | 团队研发资产 | 单次需求交付 |
| 入口 | Workspace 和 AGENTS.md | 已确认的项目与 iCafe 卡片 |
| 流程形态 | 可组合的 Skills | 固定状态机 |
| 约束方式 | Hooks、说明和团队约定 | G0 到 G9、哈希、基线和证据门 |
| 多仓处理 | 指挥仓维护仓库关系 | 每个任务绑定一个业务仓和独立测试仓 |
| 外部系统 | 按团队需要接入 | 明确接入 KU、iCafe、iCode、iPipe、如流 |
| 恢复能力 | 依赖 handoff 和各 Skill 实现 | intent、receipt、锁、心跳和持久状态 |
| 适合场景 | 建设长期团队 Agent 工作方式 | 高约束、可审计的端到端交付 |

因此没有必要在两者之间二选一。更自然的组合方式，是让 Roma Workspace 作为外层团队空间，把仓库说明、产品知识、环境约束和通用工具组织起来，再把 tom-autodev 作为其中一条受控交付路径。

`AGENTS.md` 可以规定什么需求必须走 tom-autodev。`repos/` 继续维护仓库职责、构建和测试入口。产品与设计材料留在 `docs/`。跨目录的强制限制放进 Hooks。tom-autodev 自己掌握阶段状态、审批、证据、外部系统写入和恢复。

有一个边界需要守住。同一条规则不要在 Workspace 文档、项目 Skill 和控制器里各写一份。环境和业务背景放在 Workspace，项目特有的执行约束放在项目 Skill，审批和状态转换留给 tom-autodev。否则三份内容会在不同时间过期，Agent 每次都要猜哪份算数。

## 放到 BGW 场景里看

BGW 的研发过程天然适合这种组合。Workspace 可以关联业务代码、`bgw_auto` 自动化测试和相关知识，把仓库关系、环境说明、C/C++ 约束、测试入口与排障工具整理清楚。团队成员打开同一个空间，Agent 能快速找到该读的材料和该用的能力。

真正开始一个需求时，再由 tom-autodev 接管交付。它确认 iCafe 卡片和项目，补齐验收条件，形成 Spec 与任务图，为业务仓和测试仓建立有归属的 Worktree，生成候选 Diff，完成双轴 Review，再把准确 Revision 交给 iPipe。失败后沿着证据回到 Diagnose，环境问题停在环境层，代码问题才进入修复。

Roma Workspace 让 Agent 熟悉这个团队。tom-autodev 让一次交付经得起追问。两者合起来，团队积累的知识有了执行入口，自动执行的过程也有了长期背景。

## 参考资料

- [Roma Workspace 实践文档](https://ku.baidu-int.com/knowledge/HFVrC7hq1Q/_SKPgSwp2G/jyGhbHUQQG/ZoZBrxkpSJnwWL)
- tom-autodev 的状态机、阶段协议、审批策略及 WorkspaceGate 实现

![关注微信公众号](/assets/images/ai-agent/tom-autodev-vs-roma-workspace/wechat.jpg)
