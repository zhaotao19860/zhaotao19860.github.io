---
layout: post
title: "TypeSafe Jev 接不进 AI 网关:为什么,以及它的最优用法"
date: 2026-09-20 18:00:00 +0800
categories: [AI Gateway]
tags: [TypeSafe, Jev, AI Gateway, 结构化输出]
---

![cover](/assets/images/ai-gateway/typesafe-jev-and-ai-gateway/cover-v1.png)

最近有人问我一个很具体的问题:TypeSafe 刚发布的 Jev 模型,现在有没有 AI 网关支持它?如果没有,那它到底该怎么用?

这个问题看似是"某个模型接没接入网关"的小事,实际上戳中了一个更本质的点:**并不是所有叫"模型"的东西,都能被现有 AI 网关吸收。** Jev 就是一个典型的反例。

## Jev 到底是什么

先说清楚 Jev 不是什么:它不是又一个 GPT、Claude、Gemini 那样的文本生成模型。

TypeSafe 把它定义为 "System One" 模型,做的是一件很窄但很硬的事——**快速给出软件可以直接消费的结构化决策**。你给它一段 state(状态)加上若干个有类型的问题,它并行、独立地评估每个问题,直接返回带类型的结果和概率分布,不生成文本,也不需要你去解析。

它对外暴露三种原语:

- **Choice**:从一组选项里选一个,返回 `choice` + `probabilities` + `confidence`。
- **Score**:按一个 rubric 给状态打分,返回 `score` + `probabilities` + `confidence`。
- **Noul**:判断一句陈述是不是真的,返回一个 0–1 的值。

一次 API 调用里三种问题可以混着问,每个问题彼此隔离并行评估,加问题几乎不增加响应时间,也不会有 context-rot。这套东西的设计目标,从头到尾就是"给代码用",不是"给人读"。

## 为什么现有 AI 网关接不进去

我查了 2026 年主流 AI 网关的现状——LiteLLM、OpenRouter、Portkey、Cloudflare AI Gateway、Kong、Vercel AI Gateway——它们的模型目录里**都没有 TypeSafe / Jev**。这不是巧合,而是架构上的必然。

现有 AI 网关全部建立在一个共同抽象上:**OpenAI 兼容的 Chat Completions / Responses API**,也就是"文本进、token 流出"。支持一个新的普通大模型,基本只是写个 adapter,把它的接口映射到这套 schema 上就行。

而 Jev 把这套抽象赖以成立的假设逐条打破:

- **请求形状不同**:普通模型是 `messages[]` 一段对话文本;Jev 是 `state` + 一组 typed questions,根本不是同一种入参。
- **响应形状不同**:普通模型返回文本 / 逐 token 流式;Jev 返回类型化的值、概率分布和置信度,没有 token 流可转发。
- **语义缓存对不上**:网关的 semantic cache 拿 prompt 文本或 embedding 做 key;Jev 的输入是 state+问题,缓存模型直接失效。
- **计费与观测对不上**:网关普遍按输入输出 token 计量成本和画 dashboard;Jev 不是那样按输出 token 计费的,token 维度的账单直接没意义。
- **护栏对不上**:内容审核护栏针对的是文本;Jev 输出的是结构化决策,没有可审的文本正文。
- **fallback 对不上**:网关能在 GPT 挂掉时切到 Claude,因为它们输出同构;但你没法从 Jev failover 到 GPT——一个吐结构化决策,一个吐文本,下游代码根本接不住。

一句话:支持普通大模型 = 往"文本进文本出"的目录里再加一条;支持 Jev = 网关得额外建一套"结构化决策"原语,几乎每个现成能力(缓存、计费、护栏、fallback、流式)都要重做。这就是目前没有网关做它的根本原因。

![普通模型输出文本要解析,Jev 直接输出可用的类型化决策](/assets/images/ai-gateway/typesafe-jev-and-ai-gateway/body-shape-v1.png)

## 那最优的用法是什么

既然它不适合被塞进 AI 网关,那正确的姿势是什么?我的判断是:**把 Jev 当"决策原语"直接嵌进业务代码,而不是当一个被代理的后端模型。**

具体几点:

**一、直连,别加 chat 代理层。** 从你的服务里直接调 TypeSafe 的 API/SDK。如果确实需要一个中间层,也应该是一个专门的"决策服务 / 策略层",而不是一个 chat-completions 网关。网关那套流式、prompt 模板、token 计费,对 Jev 全是负担。

**二、用它的原生原语对应你的控制流。** Choice 天然对应路由和分支,Score 对应排序和优先级,Noul 对应布尔门控。它返回的就是你 `if/switch/sort` 能直接吃的值,这才是它的价值所在——省掉了"让文本模型输出 JSON,再解析,再校验"的一整条脆弱链路。

**三、拿 confidence 做架构分流。** 这是最容易被忽略、也最值钱的一点。高置信度的决策直接自动执行;低置信度的再回退到重型 LLM 或转人工。等于用一个又快又便宜的结构化模型挡在前面,只在真正需要时才动用贵的推理。

**四、反转视角:让 Jev 当网关的"大脑",而不是网关的"后端"。** 这点对做网关的人特别关键。Jev 接不进网关的模型目录,但它非常适合做**网关自己内部的决策**——路由该走哪个上游、这条请求要不要触发护栏、语义健康探针判断某个后端是不是已经"变笨"了。也就是说,Jev 跟 AI 网关的正确关系,是"**在网关里做决策**",而不是"在网关后面被调用"。

![让 Jev 当网关的决策大脑,高置信度直接放行、低置信度回退到大模型或人工](/assets/images/ai-gateway/typesafe-jev-and-ai-gateway/body-usage-v1.png)

## 小结

Jev 之所以接不进现有 AI 网关,不是因为它不够好,而是因为它压根不是文本生成模型——它是一个结构化决策原语。想清楚这一点,用法就顺了:直连、按原语映射控制流、拿 confidence 分流,以及——如果你自己在做网关——把它请进来当决策大脑,而不是当又一个待代理的下游模型。
