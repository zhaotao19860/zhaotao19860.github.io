---
layout: post
title: "MCP 去 Session：网关终于回归标准 HTTP"
date: 2026-07-31 11:40:29 +0800
category: AI-Gateway
tags: [MCP, AI Gateway, HTTP]
excerpt: "无状态核心拆掉粘性路由，但真正的变化是把隐式会话改造成显式、可验证的状态"
---
MCP 曾把一个远程工具调用协议，运行成了网关最不喜欢的样子：先握手、再发 Session ID、后续请求还要找到原来的实例。2026-07-28 版规范终于拆掉这层隐式会话，让 MCP 重新服从普通 HTTP 基础设施的基本假设。

![](/assets/images/ai-gateway/mcp-sessionless-http/01.png)

## 7 月 29 日，协议的底层假设变了

官方仓库 \`2026-07-28\` 标签所指提交换算到北京时间是 7 月 29 日 00:44，落在本次观察窗口。当天发布的[官方变更清单](https://modelcontextprotocol.io/specification/2026-07-28/changelog)写得很直接：删除协议级 Session 与 \`Mcp-Session-Id\`，删除 \`initialize\` / \`notifications/initialized\` 握手；协议版本和客户端能力改由每个请求的 \`\_meta\` 携带。

这不是少发两个报文，而是把 MCP 的单位从“连接期间的会话”改成“自包含的请求”。服务端必须实现 \`server/discover\` 来声明版本、能力和身份，但客户端不必先调用它，也可以直接请求，在版本不兼容时根据 \`UnsupportedProtocolVersionError\` 重试。

7 月 29 日的中文技术媒体和 7 月 30 日的开发者文章迅速把焦点放到“无状态”。这个判断方向正确，但需要加一条边界：旧版服务端分配 \`MCP-Session-Id\` 是可选项。真正制造粘性路由的，是服务端一旦发出 Session ID，又把协商结果、订阅或流程上下文留在单实例内存中。

## 粘性 Session 为什么是网关的异常路径

在 2025-11-25 版 Streamable HTTP 中，客户端必须先完成初始化。服务端若返回 \`MCP-Session-Id\`，客户端后续请求就必须携带它。最省开发成本的实现会把 Session 存在本机内存，于是负载均衡器只能做 affinity：同一个 ID 必须持续命中同一实例。

这会把扩缩容、滚动发布和故障转移绑在协议状态上。实例被摘除，不只是一个请求重试，而可能是整个会话丢失。网关要么维护 Session 到实例的映射，要么让所有实例共享存储，要么解析 MCP 流量并补偿状态。原本擅长健康检查、轮询和熔断的标准 HTTP 数据面，被迫承担一套协议特有的会话生命周期。

新版将版本、能力和客户端信息放回每个请求。任何兼容实例都能独立判断并处理，请求不再因为协议 Session 被钉在某台机器上。此时普通 round-robin、least-request、无状态自动扩缩容和跨可用区故障转移重新成为默认路径。

![](/assets/images/ai-gateway/mcp-sessionless-http/02.png)

## 网关不只少了 affinity，还获得了可见路由键

[新版 Streamable HTTP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http)要求每个 POST 带 \`MCP-Protocol-Version\` 和 \`Mcp-Method\`；对 \`tools/call\`、\`resources/read\`、\`prompts/get\`，还必须带 \`Mcp-Name\`。这些 Header 是 JSON-RPC Body 关键字段的镜像，并且版本 Header 与 \`\_meta\` 不一致时，服务端必须拒绝请求。

这对网关很关键。过去若要按工具名分流、限流或审计，中间层往往需要解析 JSON Body；现在它可以直接在标准 HTTP Header 上匹配：\`tools/call\` 走高风险策略，\`resources/read\` 走数据访问审计，某个工具名绑定独立额度。\`x-mcp-header\` 还允许工具定义把指定的原始类型参数安全镜像为 Header，但客户端必须遵守字段名、类型和编码约束。

列表与资源结果也新增 \`ttlMs\` 和 \`cacheScope\`。\`public\` 结果可以由共享网关或缓存代理复用，\`private\` 结果只能在相同授权上下文中复用。MCP 没有简单照搬 \`Cache-Control\`，却开始明确拥抱 HTTP 中间层已经理解的缓存边界。

![](/assets/images/ai-gateway/mcp-sessionless-http/03.png)

## 无状态不是“没有状态”

协议删除的是隐式 Session，不是业务状态。跨调用流程仍然存在，只是必须写清楚由谁持有、如何传递、何时失效。

官方给出两条路径。普通跨调用状态使用服务端签发的显式 handle，并作为工具参数回传；需要服务端向客户端索取补充信息的多轮流程，改用 [Multi Round-Trip Requests](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/mrtr)：服务端返回 \`input_required\`，客户端收集输入后重试原请求，可携带不透明的 \`requestState\`。

这比 Session 更适合横向扩展，但也把安全责任推到明面上。规范明确要求把 \`requestState\` 当作攻击者可控输入；只要它影响授权、资源或业务逻辑，就必须用 HMAC 或 AEAD 保护完整性，并绑定用户、短 TTL 与原请求。需要严格一次性的操作仍要在服务端持久化消费记录，不能只靠一个签名 token。

SSE 也没有消失。单次请求可以返回请求作用域的 SSE，长期变化通知通过 \`subscriptions/listen\` 的响应流传递。网关仍需关闭响应缓冲、配置 idle timeout 和 keep-alive；无状态解决的是请求对实例的亲和，不是长连接运维本身。

## 迁移最容易错在三个“看起来一样”

第一，协议无状态与服务实现无状态不是一回事。若旧 Session 中保存的是购物车、审批游标、一次性凭证或长任务进度，直接删除亲和只会把偶发故障变成稳定故障。迁移前必须逐项盘点：能够由请求重建的上下文随请求携带；适合交给客户端的状态改成短期 handle；必须保证一次消费或跨天运行的状态进入持久层。

第二，独立请求与安全重试不是一回事。网络断开后，新版不再使用 \`Last-Event-ID\` 恢复 SSE；客户端要用新的 request ID 重新发起。对查询类工具这通常可行，对转账、发消息、创建资源等有副作用的工具，网关不能只看到超时就自动重放。服务端需要业务幂等键，网关需要区分“未送达”“已受理但响应丢失”和“明确失败”。

第三，Header 可路由与 Header 可盲信不是一回事。\`Mcp-Method\`、\`Mcp-Name\` 是为了让中间层快速判断，不是第二套真相来源。边缘网关可以基于它们执行粗粒度策略，终点服务仍必须校验 Header 与 JSON-RPC Body，并在不一致时拒绝。否则攻击者可能用低风险 Header 包装高风险 Body，绕过只看 Header 的授权或限流规则。

## 技术判断：MCP Gateway 将从会话代理变成策略执行点

**我的判断是，去 Session 化不会削弱 MCP Gateway，反而会淘汰它最没有差异化的一项工作：替协议维护粘性。**

当会话映射消失，通用负载均衡器可以重新负责可用性和扩缩容；MCP Gateway 的价值会集中到身份、工具级授权、风险限流、参数策略、审计、版本兼容和可观测性。\`Mcp-Method\` 与 \`Mcp-Name\` 让这些策略不必依赖深度解析，控制面也更容易把规则下发到 Envoy、NGINX、Kong 或云负载均衡数据面。

但生产迁移不能直接删除 sticky session。现代客户端与旧服务端不会自动互通，旧客户端也无法自动前进到现代协议。现实路径是先部署 dual-era 端点，观测请求版本与回退比例；再检查本地 Session 中究竟存了什么，把必要状态改造成显式 handle、受保护的 \`requestState\` 或持久任务；最后才能移除亲和规则与共享 Session Store。

网关团队还应补上四个门禁：校验 Header 与 Body 一致性；只对幂等请求自动重试；按授权上下文隔离 \`private\` 缓存；把 SSE 断流与业务取消区分开。新版规范移除了 \`Last-Event-ID\` 恢复，断开的响应流意味着在途请求丢失，客户端必须用新的 request ID 重新发起，这让幂等设计比过去更重要。

MCP 这次真正的成熟，不是从“有状态”变成“无状态”六个字，而是承认 HTTP 基础设施不该为协议的隐式连接模型买单。状态可以存在，但必须显式、可验证、可迁移；网关可以理解 MCP，但不必再被 Session 绑架。

## 参考来源

- [MCP 2026-07-28：Key Changes](https://modelcontextprotocol.io/specification/2026-07-28/changelog)

- [MCP 2026-07-28：Streamable HTTP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http)

- [MCP 2026-07-28：Versioning and Compatibility](https://modelcontextprotocol.io/specification/2026-07-28/basic/versioning)

- [MCP 2026-07-28：Multi Round-Trip Requests](https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/mrtr)

- [MCP 2025-11-25：Legacy Streamable HTTP](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)

- [IT之家：MCP 2026-07-28 规范转向无状态核心](https://www.ithome.com/0/983/102.htm)
