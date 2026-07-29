---
layout: post
title: "AI 网关与推理网关，谁会占领市场？"
date: 2026-07-29 00:00:00 +0800
categories: [AI Gateway]
tags: [AI Gateway, Inference Gateway, Kubernetes, F5]
excerpt: "答案不是二选一：统一控制面会赢得采购入口，池内推理调度会长期独立"
---

AI Gateway 能不能直接连接推理引擎，并最终吃掉 Inference Gateway？技术上可以，市场上却不会简单走向单体。真正可能成为企业标准的，是一句看似矛盾的组合：**架构分层，产品融合。**

![AI 网关与推理网关分层架构](/assets/images/ai-gateway/ai-inference-gateway-market/cover-v1.png)

## 这场争论已经从功能走向控制权

7 月 27 日，TrueFoundry 发布了[六类 AI Agent 架构及其控制要求](https://www.truefoundry.com/blog/six-ai-agent-architectures-one-control-plane)，把不同 Agent 形态背后的模型流量和工具流量拆开讨论，同时主张用统一控制面治理。这篇厂商文章不能证明市场份额，却准确暴露了企业采购中的拉力：团队不想维护更多控制台，但不同流量对象也不可能永远使用同一种调度逻辑。

围绕“AI 网关能否替代推理网关”的讨论也指向同一个矛盾。AI Gateway 管的是谁能调用、调用哪个模型、花多少钱、数据能否出域；Inference Gateway 管的是请求进入资源池以后，应该落到哪个 Pod、哪张 GPU、哪个已命中 KV Cache 或加载 LoRA 的 Worker。前者优化资源消费，后者优化资源利用率。

二者都在“路由”，但调度粒度、状态变化速度和责任人完全不同。

## 两级调度不是重复建设

一个典型企业既可能调用 Bedrock、Azure OpenAI 等外部服务，也可能拥有 NVIDIA、昇腾或其他芯片组成的自建资源池。AI Gateway 先依据身份、地域、预算、模型能力和 SLA 选择目标 Provider 或资源池。这是相对缓慢的全局决策。

进入自建池后，Inference Gateway 再根据 Queue Depth、KV Cache、GPU 负载、LoRA Adapter 和请求优先级选择实例。这些状态每秒都在变化，必须在池内就近决策。把它们全部上收给中央 AI Gateway，不仅会制造状态同步压力，也会让全局入口理解每一种推理引擎的内部细节。

[Kubernetes Gateway API Inference Extension](https://gateway-api-inference-extension.sigs.k8s.io/) 正在把这种池内协作标准化：`InferencePool` 描述模型服务池，Endpoint Picker 根据实时指标选择 Endpoint。GKE Inference Gateway 将 Gateway 数据面与 llm-d 路由智能分开；NVIDIA Dynamo 则把 KV-aware 选择逻辑接入 Endpoint Picker。它们共同说明，推理调度正在形成独立专业层，而不是普通轮询的别名。

![资源池级与实例级两级调度](/assets/images/ai-gateway/ai-inference-gateway-market/body-ai-v1.png)

## F5 给出了最有代表性的产品答案

F5 的路线值得观察，因为它没有在“合并”与“拆分”之间只选一边。

外层的 F5 AI Gateway 偏向 Prompt 与 Response 安全、模型访问治理、数据保护和企业策略；内层的 [NGINX Gateway Fabric](https://www.f5.com/products/nginx/nginx-gateway-fabric) 已支持 Gateway API Inference Extension，官方明确列出按模型类型、版本、成本性能档位、Cache 和 Queue Depth 路由。这已经是 Inference Gateway 的工作。

但在商业包装上，F5 又通过 NGINX One 把 NGINX Plus、Kubernetes Ingress、Gateway API、WAF、集中管理和推理感知流量放进统一体系。其隐含判断不是“只能部署一层”，而是“客户应该购买一个平台，再按位置部署不同网关角色”。

[Envoy AI Gateway 的参考架构](https://aigateway.envoyproxy.io/blog/envoy-ai-gateway-reference-architecture) 也采用类似思路：Tier One Gateway 负责统一入口、外部 Provider 和粗粒度策略；Tier Two Gateway 位于自建模型服务集群，负责本地流量与推理池。底层都可以使用 Envoy，但故障域和职责并不相同。

![统一控制面与分层数据面](/assets/images/ai-gateway/ai-inference-gateway-market/body-fact-v1.png)

## 统一产品为什么不会变成单体数据面

统一控制面有明显收益：身份和模型目录只维护一次，预算与合规策略使用同一版本，跨层 Trace 可以在一个界面关联，采购也不必为每个资源池重复谈判。但这并不要求所有请求都穿过一台中央代理。控制面可以把策略编译并下发给不同位置的数据面，运行时则让请求在完成全局选择后直接进入目标池，避免中央节点承担全部高速状态。

单体数据面的真正问题不是多一项功能，而是故障和演进被绑在一起。企业修改 Prompt 安全策略时，不应该迫使 GPU 调度器同步升级；推理团队替换 KV Router 或上线新的 LoRA 调度算法时，也不应该改变外部模型契约。两层拥有独立发布节奏，才能让安全治理和性能优化各自快速演进。

同样需要警惕相反极端。若两层都执行认证、Token 限流、重试、缓存和 Fallback，请求可能被重复扣费，失败时产生重试放大，流式取消信号也可能在中间丢失。分层成立的前提不是组件数量，而是每项策略只有一个决策者：AI Gateway 决定能否调用以及去哪个池，Inference Gateway 只决定池内落点；两层共享身份与观测上下文，但不争夺同一个策略所有权。

## 哪种形态会占领哪一部分市场

只使用公有云模型 API 的团队，仍会选择单层 AI Gateway。云厂商内部当然存在复杂推理调度，但它对客户不可见；客户只需要模型目录、虚拟密钥、预算、审计、Fallback 和安全策略。

只有少量 vLLM 或 SGLang 实例的团队，也会倾向融合部署。此时再增加一个同步代理，带来的故障点和运维成本可能高于 GPU 调度收益。

真正决定高价值市场的是大规模自建与混合推理。只要企业同时拥有多地域、异构芯片、多个推理引擎、大量 LoRA 或 Prefill/Decode 分离，两级调度就会成为经济必然。原因很直接：网关多消耗几台 CPU 的成本有限，而昂贵加速卡的利用率、TTFT 和排队效率会直接决定推理账单。

因此，按请求数量看，单层架构可能长期占多数；按企业基础设施投入和产品收入看，分层架构会占据更高价值市场。

## 技术判断：最终赢家是“可合可分”

**我的判断是，最终占领市场的不是一个无所不包的超级网关，也不是两套互不相干的产品，而是统一控制面加分层数据面。**

统一控制面负责身份、模型目录、预算、合规、策略版本和全链路观测；AI Gateway 数据面负责准入与跨池路由；Inference Gateway 数据面负责池内 Endpoint 选择；推理引擎继续负责 Batch、KV Block 和 GPU 计算。小规模时，两个数据面角色可以合并；规模扩大后，可以在不更换控制体系的前提下拆开。

这也决定了厂商竞争的下一阶段。传统 API、ADC 和安全厂商掌握企业入口与采购关系，推理平台和芯片厂商掌握 KV Cache、Worker 与 GPU 优化。任何一方都很难完整吞掉另一方。更可能出现的是开放接口上的合作：上层选择资源池，下层通过 Gateway API Inference Extension 一类标准完成实例调度。

对企业架构团队，当前最务实的动作不是立刻增加第二跳，而是先把边界定义清楚：全局层只依赖资源池级信号，池内层拥有快速变化的推理状态；鉴权、限流、重试和缓存只能有一个明确责任方；Trace ID、取消信号和流式响应必须贯穿两层。满足这些条件，架构才能从单层平滑演进，而不会重演 ESB 式的集中化失控。

市场终局可以浓缩为一句话：**客户只想看见一个平台，但系统内部必须允许两种网关继续专业化。**

## 参考来源

- [TrueFoundry：Six AI Agent Architectures—and the Controls Each One Needs](https://www.truefoundry.com/blog/six-ai-agent-architectures-one-control-plane)
- [F5 NGINX Gateway Fabric](https://www.f5.com/products/nginx/nginx-gateway-fabric)
- [Kubernetes Gateway API Inference Extension](https://gateway-api-inference-extension.sigs.k8s.io/)
- [Envoy AI Gateway Reference Architecture](https://aigateway.envoyproxy.io/blog/envoy-ai-gateway-reference-architecture)
- [GKE Inference Gateway powered by llm-d](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/about-gke-inference-gateway)
- [NVIDIA Dynamo Gateway API Inference Extension](https://docs.nvidia.com/dynamo/dev/kubernetes-deployment/request-routing/gateway-api-inference-extension/overview)
