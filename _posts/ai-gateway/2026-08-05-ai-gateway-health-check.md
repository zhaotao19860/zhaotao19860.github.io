---
layout: post
title: "AI 网关如何做健康检查"
date: 2026-08-05 15:24:30 +0800
category: AI-Gateway
tags: [AI Gateway, Health Check]
excerpt: "从端口探活走向模型可服务性判断，三层信号决定流量该不该继续送"
---
![](/assets/images/ai-gateway/ai-gateway-health-check/01.png)

很多网关的健康检查，起点是一只很小的探针。它向上游发一个 TCP SYN，或者请求一个 HTTP 路径，连续失败几次就把节点摘掉。对普通 API 来说，这已经能解决一大半问题。到了大模型服务，探针拿到一个 200，事情仍然可能很糟。

模型进程还活着，GPU 队列却已经排满。请求能建立连接，首个 token 却要等很久。模型返回了内容，错误率和限流次数也正在抬头。把这些状态都压成一个“健康”布尔值，路由器得到的结论会比真实情况简单得多。

我把公开文档和实现对了一遍，比较稳妥的做法是把健康检查拆成三层。第一层确认进程和网络还在工作，第二层确认这条模型能力真的能调用，第三层根据真实请求观察延迟、错误和容量。三层信号最后汇成路由权重、熔断和回退决策。

## 第一层，确认它还在

L4 检查回答的问题很窄。端口是否监听，TCP 是否能握手，网络路径是否可达。它适合发现进程退出、端口关闭、安全组错误和明显的网络故障。

L7 检查再向前走一步。网关定期请求一个专用 HTTP 路径，检查响应码和超时。阿里云 AI 网关文档把 TCP 检查用于端口存活，把 HTTP 检查用于应用状态，并用连续成功次数和连续失败次数控制状态切换。Kong Gateway 也把主动检查定义为对指定 HTTP 或 HTTPS 路径的周期性请求。

这里有两个容易被忽略的参数。检查间隔决定探针多久来一次，阈值决定连续几次结果才改变状态。间隔很短，故障发现更快，也会增加探针流量。阈值很低，恢复很快，也更容易在网络抖动时反复摘除节点。健康检查本身是一种有成本的控制动作。

探测路径还要单独设计。它不应触发完整推理，不应消耗大量 token，也不应因为业务鉴权、日志写入或下游数据库短暂异常而误判模型进程。进程级 \`/health\` 和接流量前的 \`/ready\` 最好分开。LiteLLM 的 Kubernetes 示例也把代理存活和就绪探针分成两个端点。

## 第二层，确认模型真的能服务

端口活着以后，网关还需要问一个更接近用户的问题。这个模型接口能否完成一次最小、可控、可计量的调用。

这类探测可以使用与生产请求相同的协议，但输入要极小，输出上限要明确，结果只判断协议、认证、模型能力和响应格式。聊天模型走 chat completion，embedding 模型走 embedding，语音模型走对应的 API。LiteLLM 的模型健康检查会根据 \`model_info.mode\` 选择探测操作，这个细节很关键。一个通用 \`/health\` 无法证明所有模型模式都能工作。

探测也不能只看 HTTP 状态码。至少要记录 DNS、连接、TLS、首字节、首 token、完整响应和错误类别。\`401\` 多半是凭据或权限问题，\`429\` 可能是配额或限流，\`5xx\` 更接近供应商或服务端故障，超时还要区分排队超时和生成超时。它们都能让请求失败，处理方式却不同。

健康状态最好有 \`healthy\`、\`degraded\`、\`unhealthy\` 三档。\`degraded\` 节点可以保留少量流量，用来验证恢复，也可以只接低优先级请求。直接二元摘除会丢掉很多信息，所有节点都变差时还会把剩下的节点压垮。

## 第三层，观察真实负载

真正有价值的健康信号，常常来自业务请求。APISIX 的 AI 网关文档把 TTFT、令牌使用量和错误率列为重要观测指标，\`ai-proxy-multi\` 还提供负载均衡、回退和健康检查。它们说明 AI 网关的健康判断已经从“能不能返回”扩展到“返回得是否值得继续使用”。

对流式对话，TTFT 是最先该看的指标。它高，常常意味着供应商排队、冷启动或 GPU 资源紧张。首 token 之后还要看 ITL，也就是 token 之间的间隔。TTFT 正常而 ITL 变差，用户依然会觉得模型卡住。两者不能用一次总耗时代替。

自托管推理还需要加入队列深度、KV cache 使用率、并发请求数和模型能力标签。Kubernetes Gateway API Inference Extension 将 Inference Gateway 与 Endpoint Picker 结合，让调度器根据 Model Serving 提供的 Metrics and Capabilities 选择副本。KV cache 状态、请求成本和 LoRA adapter 能力都可以成为路由依据。轮询只知道节点数量，推理网关需要知道每个节点此刻能接什么请求。

被动检查的价值在这里最明显。Envoy 的 outlier detection 可以按连续 5xx、网关失败、成功率和失败百分比识别异常上游，再用隔离时间和最大剔除比例限制影响范围。Kong 的文档提醒，被动检查只负责把异常 Target 暂时禁用，自动恢复仍需要主动检查配合。

![](/assets/images/ai-gateway/ai-gateway-health-check/02.png)

## 各家实现怎么取舍

开源软件的差别，主要落在健康信号由谁提供，以及摘除后由谁恢复。

Envoy 把主动检查和 outlier detection 分开。前者按集群配置探测，后者从真实流量里统计连续 5xx、网关失败和成功率异常。它适合做精细的数据面治理，代价是需要外部控制面接入模型、租户和成本信息。

Kong 的 active check 能自动禁用和重新启用 Target，passive check 更像 circuit breaker，发现请求错误后摘除节点。它适合 API 团队快速治理上游，但多模型场景仍要额外处理模型映射、供应商错误和流式回退。

APISIX 同时提供 active 和 passive check，\`ai-proxy-multi\` 还把多供应商转换、重试、回退和检查放在同一插件里。它的好处是接入路径短，Prometheus 也能看到上游状态。限制同样写在官方文档里，部分 LLM 供应商没有官方 health endpoint，网关只能用自定义路径或真实请求信号补足。

LiteLLM 的思路更靠近模型路由。它可以开启后台检查，把失败部署移出路由池，并用 \`AllowedFails\` 和 \`cooldown_time\` 区分认证错误、超时和限流。它适合多供应商聚合，代价是探测会消耗额度，还需要共享状态避免多副本重复探测。

Kubernetes Gateway API Inference Extension 和 GKE Inference Gateway处理的是另一类问题。它们让 L7 代理把请求交给 Endpoint Picker，再按 KV cache、等待队列、GPU 负载、prefix cache 和 LoRA adapter 选择副本。性能收益明显，系统也更复杂，适合自托管推理集群，不适合只想代理几个外部模型 API 的团队。

云厂商通常把复杂度收进控制台。阿里云 AI 网关提供主动检查、被动失败率和恐慌阈值。Azure API Management 的 backend pool 叠加 circuit breaker，按失败次数、状态码和时间窗口暂停后端，等待 trip duration 到期再恢复。它们部署快，缺点是健康模型和回退细节受产品边界约束，跨云迁移时要重做配置。

## 怎么选

- 只有 HTTP API 和少量副本，选 APISIX、Kong 或 Envoy。需要完整 API 管理选 APISIX 或 Kong，需要可编程数据面和外部控制面选 Envoy。

- 有多个商业模型和供应商，选 LiteLLM 或带模型治理能力的云 AI 网关。重点验证 429、认证失败和供应商限流是否能分别处理。

- 自建 vLLM、SGLang 或 GPU 集群，选 Inference Gateway 路线。健康检查要接队列和 KV cache，否则只是在 GPU 服务前面加了一层普通负载均衡。

- 已经深度使用单一云平台，优先用该平台的 circuit breaker 和监控，再补模型级探测。

- 需要跨云或长期保持可迁移，保留 Envoy、APISIX 或 LiteLLM 作为统一入口，把厂商探测适配在 provider 层。这样更灵活，运维和测试成本也会随之增加。

我的选择标准只有一句。看团队能拿到什么健康信号，再看故障发生后谁负责恢复。没有队列、TTFT 和配额数据时，先做好主动检查与被动隔离；有了这些数据，再升级到模型感知的推理调度。

![](/assets/images/ai-gateway/ai-gateway-health-check/03.png)

## 健康结果怎样进入路由

健康检查如果只更新控制台颜色，作用很有限。它应当改变候选列表和每个候选的分数。一个简单的决策顺序如下。

- 先过滤协议不通、认证失败、模型能力不匹配的节点

- 再按租户、数据合规、模型版本和工具能力筛选

- 在剩余节点中比较队列、TTFT、ITL、错误率和成本

- 对 \`degraded\` 节点降低权重，对连续失败节点执行隔离

- 流式请求在首 token 到达前允许回退，开始输出后记录中断并结束本次连接

最后一点常被忽略。流式响应一旦已经把响应头和部分内容发给客户端，网关无法把已经发送的内容无缝搬到另一个供应商。回退窗口只存在于真正开始输出之前。重试也要带请求幂等性、token 成本和副作用判断，不能看到 502 就无条件再打一遍。

## 我的判断

AI 网关的健康检查，应该从一个布尔值变成一组有时效的证据。L4 证明网络和进程，L7 证明协议和模型能力，真实流量证明当前负载下的可服务性。三层信号互相补足，也互相纠错。

实施时可以先做四件小事。给每种模型模式配置独立的就绪检查，记录 TTFT 和 ITL 的分位数，把错误拆成认证、限流、上游服务和本地超时，最后为摘除和恢复设置不同的连续阈值。阈值不要照抄产品默认值，先用自己的流量做基线，再验证一次节点故障、配额耗尽、GPU 排队和供应商降级。

健康检查的终点不是把异常节点藏起来。它要让网关知道何时继续送流量，何时减量，何时换路由，以及何时承认所有候选都在变差。这个判断越接近真实请求，AI 网关就越像一个可靠的流量系统。

## 来源

- [阿里云 AI 网关服务健康检查](https://help.aliyun.com/zh/api-gateway/ai-gateway/user-guide/configure-service-health-checks)

- [Kong Gateway 健康检查与熔断](https://developer.konghq.com/gateway/traffic-control/health-checks-circuit-breakers)

- [Envoy Outlier Detection](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/outlier)

- [APISIX AI 网关介绍](https://apisix.apache.org/zh/blog/2025/04/08/introducing-apisix-ai-gateway)

- [Kubernetes Gateway API Inference Extension](https://github.com/kubernetes-sigs/gateway-api-inference-extension)

- [LiteLLM Health Checks](https://docs.litellm.ai/docs/proxy/health)

- [Azure API Management backends](https://learn.microsoft.com/en-us/azure/api-management/backends)

![](/assets/images/ai-gateway/ai-gateway-health-check/04.jpeg)
