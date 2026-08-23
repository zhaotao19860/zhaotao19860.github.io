---
layout: post
title: "十二个 PR 比五份榜单更能说明自托管 AI 网关的成色"
date: 2026-08-02 19:34:30 +0800
category: AI-Gateway
tags: [AI Gateway, Open Source]
excerpt: "7 月 31 日 LiteLLM 的一个 dev 版里，改的全是计量、限流并发、流式检查和工具身份边界——没有一条是关于支持更多模型的"
---
![](/assets/images/ai-gateway/self-hosted-ai-gateway-pr-review/01.png)

## 榜单在比排名，代码在补别的东西

7 月 30 日凌晨，Maxim AI 发了一份 [Top 5 Open Source AI Gateways in 2026](https://www.getmaxim.ai/articles/top-5-open-source-ai-gateways-in-2026-for-enterprise-ai/)，Bifrost 排第一，LiteLLM 第二。Maxim AI 就是 Bifrost 的开发方。同一天下午，Pinggy 发了篇 [OmniRoute 实测](https://pinggy.io/blog/omniroute_ai_gateway_security/)，主角是个 2 月中旬才建仓、现在 37,120 星的自托管网关。

这两篇都不在我这次的两日窗口（07-31 至 08-01）里，它们卡在窗口前一天。但正因为差了一天，对照才有意思：窗口外那天在比排名、星数和聚合了多少家供应商；窗口内这两天，同一个赛道在改的是完全另一类东西。

## 窗口里真正发生的事

7 月 31 日 14:46，LiteLLM 打出 [v1.96.0-dev.2](https://github.com/BerriAI/litellm/releases/tag/v1.96.0-dev.2)。这是 prerelease，不是 GA，变更日志 23 项。我按 GitHub API 逐条核了 \`merged_at\`，其中十二个 PR 在这两天内合入。几乎没有一条是关于"支持更多模型"的。

![](/assets/images/ai-gateway/self-hosted-ai-gateway-pr-review/02.png)

## 第一类：这笔钱算在谁头上

[\#35290](https://github.com/BerriAI/litellm/pull/35290) 让代理默认向上游索取 stream usage，再从返给客户端的流里把它剥掉。

要理解这为什么是必须做的事，得看非流式和流式的差别。非流式响应里 \`usage\` 天然带在响应体里，网关顺手就能记账。流式不给——除非你显式要：

    {"stream": true, "stream_options": {"include_usage": true}}

不要，最后一个 chunk 里就没有 \`usage\`，网关想计费只剩两条路：自己重新 tokenize 一遍去估，或者干脆不记。前者会和上游的实际计费口径偏离，后者等于放弃预算和配额。所以网关必须替客户端把这个开关打开——可打开之后，客户端会多收到一个它没申请过的 chunk，行为就开始依赖上游的实现细节。剥离这一步不是洁癖，是让客户端契约保持稳定。网关在这里同时是知情方和过滤器。

同批里 [\#35300](https://github.com/BerriAI/litellm/pull/35300) 给 auto-router 自己的 classifier 调用打标，[\#35320](https://github.com/BerriAI/litellm/pull/35320) 让 fast service tier 按 priority 费率计费，[\#35292](https://github.com/BerriAI/litellm/pull/35292) 把 \`litellm_metadata\` 改成按引用绑定，好让 guardrail 信息进得了 spend logs。四个 PR 在回答同一个问题：谁花的钱，算在谁头上。

其中 auto-router 那条值得单独看一眼。语义路由本身要先调一次小模型做分类，这次调用是真花钱的，但它不是用户请求的一部分。不打标，它就会被摊到某个用户账上；打了标，路由这层的开销才第一次变成可以单独看的一行。这类问题只有真正开始给人开账单之后才会被问到。

## 第二类：限流状态该住在哪里

[\#35278](https://github.com/BerriAI/litellm/pull/35278) 把 v3 限流器的 per-request 暂存从 request metadata 挪到 ContextVar。标题写的是 \`refactor\`，实质是并发正确性的位置问题。

差别在归属。挂在请求元数据上的中间状态，本质是一个被整条调用链共享的可变字典：路由、重试、fallback、日志回调都能碰它，谁清、什么时候清，取决于约定而不是机制。一旦某一层忘了清，或者一个请求对象在重试路径上被复用，状态就跨请求活下来了。\`ContextVar\` 的语义不一样——它绑在异步上下文上，\`asyncio\` 创建新任务时拷贝当前上下文，子任务的写入不会回灌父上下文。也就是说边界由运行时给，不由调用方的自觉给。

多租户网关的限流一旦串了租户，是最难在测试里看见、最容易在生产里出事的那一类 bug——它不报错，只是让某个 key 的配额莫名其妙地多了或少了。单请求的单元测试永远测不出来，因为它只在并发交错时出现。

![](/assets/images/ai-gateway/self-hosted-ai-gateway-pr-review/03.png)

## 第三类：流式响应上怎么做后置检查

[\#35260](https://github.com/BerriAI/litellm/pull/35260) 让 \`/v1/messages\` 的流式响应也跑 post_call guardrail。流式和后置检查天然冲突：后置检查要看完整输出才能判断，可 token 已经一路发给客户端了。能选的只有三种做法——全缓冲到底再放行，等于放弃流式；边发边查，那么违规内容已经出去了，只能中断在半句话上；或者滑窗延迟若干 chunk 再放行，用一点首字延迟换一个可撤回的窗口。三种都有代价，没有免费选项。

这个 PR 的价值不在选了哪种，而在它走统一 guardrail 翻译层，检查逻辑不再按端点各写一份。这是从"某个端点上挂了 guardrail"走向"guardrail 是一层"的分界。做过的人都知道，\`/chat/completions\` 上装好了、\`/v1/messages\` 上漏了，是这类系统最典型的失效方式。

配套的 [\#35259](https://github.com/BerriAI/litellm/pull/35259) 让配置里定义的 guardrail 不依赖数据库也能通过 list/info 端点取到，且 id 稳定；[\#35263](https://github.com/BerriAI/litellm/pull/35263) 让 config 定义的 policy 在 DB 同步之后不被冲掉。这两条合起来是一件事：策略的真源到底在配置文件还是数据库。id 不稳定意味着审计日志里的引用会在重启后失效；config 被 DB 同步冲掉意味着你 GitOps 管的策略会静默消失。同批的 [\#35294](https://github.com/BerriAI/litellm/pull/35294) 还修了一个更细的 bug——上下文裁剪时不再压掉模型必须响应的那一轮，这是"为了塞进窗口而把要点删了"的经典事故。

## 一个 200 掩盖的错误，客户端永远重试不了

[\#35307](https://github.com/BerriAI/litellm/pull/35307) 把流内错误码全量映射到真实 HTTP status。这条容易被当成杂项，但它决定了上层能不能自动恢复。

流式响应的头在第一个 chunk 之前就发出去了，此时状态码已经是 200。之后上游报错，错误只能以事件的形式出现在流里。对客户端而言，这是一个"成功"的响应，只是内容里藏了个错误对象——重试中间件不会触发，熔断器不会计数，SLO 面板上这就是一次正常请求。网关如果不做映射，整条链路的错误可观测性都是假的。

## 第四类：代理工具，认证边界就得重画

[\#34856](https://github.com/BerriAI/litellm/pull/34856) 是这批里唯一带 \`!\` 的：keyless gateway 的 OAuth 流程扩展到 per-server 的 MCP URL path。

网关一旦开始代理工具而不只是模型，认证边界就得按 server 切。原因是权限的性质变了：代理模型时，网关握的是一把对称的 API key，能力边界就是"能不能调这个模型"。代理 MCP server 时，每个 server 背后是一套不同的外部授权——一个连着代码仓库，一个连着工单系统，一个连着数据库。用一份网关级凭据统一代理，就成了典型的 confused deputy：调用方只被授权用工单工具，却借网关的身份摸到了仓库写权限。按 server 切 OAuth，是把这条边界从"网关自己记得别越权"变成协议层的事。

conventional commit 的 \`!\` 是我能确认的破坏性信号，具体迁移成本官方没有单独说明。顺带一提，同批的 [\#35291](https://github.com/BerriAI/litellm/pull/35291) 给 S3 两条日志路径都加上了 SSE-KMS——这是合规审计会逐条问的那种要求，不是功能亮点。

## 那两篇里的数字，为什么不能用来选网关

Bifrost 的"5,000 RPS 下每请求 11 微秒开销"，条件写在它 [自家的 benchmark 文档](https://docs.getbifrost.ai/benchmarking/getting-started) 里：全部基于 mocked OpenAI 调用；11μs 是 t3.xlarge 的数，t3.medium 上是 59μs；端到端平均 1.61s 中 HTTP Request 占 1.50s。仓库侧栏自己给的是更松的 \`\<100 µs\`。README 写 23+ providers，"1,000+ models"只出现在侧栏描述、正文里没有。Bifrost 现在 6,962 星，Apache-2.0。

OmniRoute 那边，Pinggy 文里的 33,908 星和 v3.8.48 在它自己发布时就已过期——v3.8.49 比该文时间戳早十一个小时。项目对外的 provider 数在 README（290+）和 wiki（250）之间不一致。Pinggy 自己倒很诚实：探测的七个免认证供应商池里只有两个有响应。

Maxim 文中引的 a16z「37% 企业生产已跑 5+ 模型」，我没能溯源到 a16z 原文，所以这篇不拿它立论。

## 技术判断

**自托管 AI 网关确实在从玩具变标配，但把这件事证明出来的不是榜单、星数或聚合供应商数，而是上面十二个 PR 的类型分布。** 一个网关值不值得进生产，取决于它对四类问题有没有答案：断连时的账怎么算、限流状态归谁、流式响应怎么检查、策略的真源在哪。这些维度在任何榜单的评分表里都不出现，因为它们只有在真正跑起多租户之后才会被问到。

限度也说清楚。dev 版不是 GA；一个两日切片证明不了趋势，只能证明这个项目在这两天的注意力落在哪。而 LiteLLM 被 [Northflank 07-29 的榜单](https://northflank.com/blog/best-open-source-ai-gateways) 排第一、被 Maxim 07-30 的榜单排第二，这本身就说明榜单之间并不一致。榜单也不是没用——它对"有哪些候选"有效，只是对"该选哪个"无效。

## 落到动作

- 拿"客户端中途断连时这笔账怎么记"去问你正在评估的网关。这是最快分辨成熟度的一题。

- 检查你的限流器把 per-request 状态挂在哪。挂在请求元数据上，就要确认它在异步链路上不会串租户。

- 如果你在流式接口上做内容检查，确认它对每个端点都生效，而不只是 \`/chat/completions\`。

- 把 config 与 DB 的策略优先级写进文档。这是升级时最容易静默丢配置的地方。

- 下次读到"X 微秒开销"，先看上游是不是 mock、机型是什么。

## 来源

- LiteLLM v1.96.0-dev.2 release：https://github.com/BerriAI/litellm/releases/tag/v1.96.0-dev.2

- 上述 PR 的 \`merged_at\` 均取自 GitHub API \`/repos/BerriAI/litellm/pulls/{n}\`

- Bifrost benchmark 方法与条件：https://docs.getbifrost.ai/benchmarking/getting-started

- Maxim AI 榜单：https://www.getmaxim.ai/articles/top-5-open-source-ai-gateways-in-2026-for-enterprise-ai/

- Northflank 榜单：https://northflank.com/blog/best-open-source-ai-gateways

- Pinggy OmniRoute 实测：https://pinggy.io/blog/omniroute_ai_gateway_security/

- 星数、license、建仓时间取自 GitHub API，查询时间 2026-08-02
