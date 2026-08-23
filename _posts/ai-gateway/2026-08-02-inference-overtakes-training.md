---
layout: post
title: "推理首超训练，AI 网关迎来高光"
date: 2026-08-02 19:19:55 +0800
category: AI-Gateway
tags: [AI Gateway, Industry]
excerpt: "算力竞争从造出模型转向持续交付 Token，网关开始接管延迟、容量与成本的实时决策"
---
AI 基础设施的重心正在移动：过去最醒目的是一次次更大的训练集群，接下来更难管理的，却是每天持续发生、直接面对用户的推理请求。

![](/assets/images/ai-gateway/inference-overtakes-training/01.png)

## 206 亿美元背后的分水岭

Gartner 在 2025 年 10 月发布的预测显示，全球 AI-optimized IaaS 终端用户支出将在 2026 年达到 375 亿美元，其中推理工作负载为 206 亿美元，占 55%，首次超过训练密集型工作负载。按总额减去推理支出计算，训练约为 169 亿美元。到 2029 年，推理占比预计将超过 65%。

这里需要强调两层边界。第一，这是预测，不是已经发生的支出。第二，公开新闻稿可以核验 375 亿、206 亿和占比，但没有直接列出“2029 年推理 717 亿美元”，因此本文不采用这个二次传播数字。

真正值得关注的也不是一条曲线超过另一条，而是基础设施的优化目标发生了变化。训练像大型工程项目：任务集中、周期相对明确，可以围绕吞吐、集群规模和完成时间优化。推理则像持续运营的公共服务：请求随时到达，输入长度不同，优先级不同，还要同时面对首 Token 延迟、每 Token 延迟、可用性和成本约束。

![](/assets/images/ai-gateway/inference-overtakes-training/02.png)

## 推理不是把训练集群换个入口

同一模型的两个请求，消耗可能相差几个数量级。一个简短分类请求和一段长上下文 Agent 任务，不能只按“请求一次”计量；后者还可能触发多轮模型调用、工具调用和重试。传统负载均衡器看到的是 HTTP 连接，推理系统必须看到模型、Token、队列和缓存。

这正是 AI 网关身价上升的原因。它不只是把 OpenAI-compatible 请求转发给后端，而要回答四个实时问题：这个调用是否允许；应该选择哪个模型或 Provider；进入自建资源池后应该落到哪个实例；这次调用该计入谁的预算与 SLO。

其中前两项偏向 AI Gateway：身份、配额、地域、模型能力、内容策略和跨 Provider fallback。第三项属于 Inference Gateway：它需要知道某个 Worker 是否已加载目标模型或 LoRA Adapter，队列有多长，KV Cache 是否命中，以及 GPU 当前是否拥塞。最后一项把两层重新连起来：只有贯通请求身份、路由决策、Token 用量和业务结果，FinOps 才不是一张月底账单。

## Endpoint Picker 把调度拉进请求路径

Kubernetes Gateway API Inference Extension 给出了一种清晰实现。\`InferencePool\` 描述一组模型服务端点；Gateway 收到请求后，将必要信息交给 Endpoint Picker；后者结合模型服务器指标选择 Endpoint，再由 Gateway 完成转发。

GKE Inference Gateway 的 llm-d Endpoint Picker 已把信号具体化：KV Cache 利用率、队列长度、前缀缓存状态和 LoRA affinity 都可以参与实例评分。普通轮询只追求请求数量平均，推理感知路由追求的是计算工作量更少、等待时间更短、已有状态尽量复用。

![](/assets/images/ai-gateway/inference-overtakes-training/03.png)

这也解释了为什么“贴近用户部署”只是结论的一半。对交互式语音、在线搜索和实时 Agent，网络往返时延确实会推动推理资源靠近需求；但批量摘要、离线评测和高吞吐生成可能更看重加速器价格与批处理效率。网关真正的价值不是无条件把请求送到最近机房，而是在延迟、数据驻留、可用容量、缓存命中和单 Token 成本之间动态选择。

## Token 经济性成为新的仪表盘

Gartner 在 2026 年 5 月进一步提出，AI 基础设施应关注 \`Token/Dollar\`、\`Token/Watt\` 和 \`Token/Second\`。这三个指标分别对应成本、能效和吞吐，却不能独立优化。为了提高每秒 Token 而盲目扩大 Batch，可能伤害交互延迟；为了压低每 Token 成本而总选小模型，可能降低任务成功率，最终触发更多重试。

因此，网关不能只做流量工程，还要参与经营闭环。路由策略至少需要同时观察：按业务划分的质量门槛、端到端延迟、缓存收益、模型单价、加速器利用率，以及失败后的实际重试成本。最便宜的一次调用，不一定带来最低的任务总成本。

## 普通 API 指标为什么不够

传统网关常用 QPS、状态码和 p99 延迟描述服务，但这些指标无法完整解释推理效率。两个都返回 \`200\` 的请求，可能一个首 Token 很快却在生成阶段卡顿，另一个因长上下文预填充而等待较久，随后输出稳定。把二者压成一次 HTTP 延迟，会掩盖真正瓶颈。

推理链路至少要拆开 TTFT、Inter-Token Latency、输入输出 Token 数、排队时间和缓存命中。对 Agent 任务还要记录一次业务目标触发了多少次模型与工具调用，以及最终是否成功。否则，路由器可能在单次调用上省钱，却因为模型能力不足造成规划失败和重复尝试，任务总成本反而上升。

缓存也不是命中越多越好。精确缓存适合确定性、高重复请求；语义缓存需要承担 embedding、向量检索和错误复用的代价；KV Cache 复用减少的是模型端重复计算，并不等于可以直接返回旧答案。这三类缓存处在不同层次，若网关只汇报一个“缓存命中率”，平台团队很难判断节省来自哪里，质量风险又落在谁身上。

更可靠的做法是给每条路由策略绑定业务目标。例如，客服场景关注首响、解决率和每个已解决会话成本；代码 Agent 关注任务成功率、工具调用次数和总 Token；批量摘要则更适合观察吞吐与每百万 Token 成本。只有指标与业务结果对齐，AI 网关才有依据在大模型、小模型、云 API 和自建推理池之间做选择。

## 技术判断：高光属于决策层，不只是代理层

**我的判断是，推理支出超过训练，不会自动让所有 AI 网关厂商受益；真正受益的是能把路由决策与 Token 经济性闭环的产品。**

只提供协议转换、虚拟 Key 和简单轮询的网关会逐渐商品化。更有价值的控制点会向两端延伸：向上连接身份、预算、质量评估和合规策略，向下连接队列、KV Cache、LoRA、GPU/NPU 状态和 Endpoint Picker。它不必亲自执行所有调度算法，但必须让策略、实时信号和计量结果在同一条请求链上可追踪。

对平台团队，眼下可以做三件具体的事。第一，把成本口径从“每次请求”改为按模型、输入输出 Token、缓存命中和业务任务归因。第二，区分跨模型或跨资源池路由与池内实例选择，避免两层同时重试、限流或缓存。第三，用真实流量同时压测 TTFT、任务成功率和每个成功任务的总成本，而不是只看单模型吞吐榜单。

训练决定模型能达到多高，推理决定模型能否被持续、可靠、经济地使用。当后者成为更大的支出项，网关就不再只是入口，而是 AI 生产系统每天都要经过的成本与体验决策点。

## 参考来源

- [Gartner：AI-Optimized IaaS Is Poised to Become the Next Growth Engine](https://www.gartner.com/en/newsroom/press-releases/2025-10-15-gartner-says-artificial-intelligence-optimized-iaas-is-poised-to-become-the-next-growth-engine-for-artificial-intelligence-infrastructure)

- [Gartner：塑造 AI 基础设施未来的三大主要技术趋势](https://www.gartner.com/cn/newsroom/press-releases/2026-0527-gartner-identifies-3-major-tech-trends-shaping-the-future-of-ai-infrastructure)

- [Kubernetes Gateway API Inference Extension](https://gateway-api-inference-extension.sigs.k8s.io/)

- [GKE Inference Gateway](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/about-gke-inference-gateway)
