---
layout: post
title: "A2A 的 agent 发现：路径钉死在 /.well-known/agent-card.json"
date: 2026-09-02 20:04:36 +0800
categories: [AI-Agent]
tags: [A2A, AI Agent, Agent Card, RFC 8615, 服务发现]
excerpt: "A2A 把 agent 发现压成了一次 GET。路径由 RFC 8615 的 well-known 前缀钉死，知道域名就够了，不用先问目录。但它只覆盖了一半"
---

![cover](/assets/images/ai-agent/a2a-well-known-agent-card/cover-v1.png)

A2A 要解决的第一个问题不是通信，是「两个 agent 怎么互相找到」。它的答案很朴素：把自描述文件放在一个所有人都提前知道的位置。

## 一、路径为什么可以钉死

`/.well-known/` 这个前缀是 [RFC 8615](https://datatracker.ietf.org/doc/html/rfc8615) 定的保留命名空间，用途本来就是「客户端不需要被告知位置就能取到的元数据」。A2A 直接沿用，把 agent 名片钉在：

```text
https://{domain}/.well-known/agent-card.json
```

这一步省掉的东西是「目录」。没有注册中心、没有 discovery 接口、不用先取一个 index 再从里面查路径。知道域名就够了，剩下的是一次 HTTP GET：

```bash
curl -s https://smart-thermostat.example.com/.well-known/agent-card.json
```

## 二、取回来的是什么

这份 JSON 叫 Agent Card。v1.0 的最小形态：

```json
{
  "name": "Invoice Reconciliation Agent",
  "version": "3.1.0",
  "supportedInterfaces": [
    {
      "url": "https://agents.northwind.example.com/a2a",
      "protocolBinding": "JSONRPC",
      "protocolVersion": "1.0"
    }
  ],
  "provider": { "organization": "Northwind Finance" },
  "capabilities": { "streaming": true, "extendedAgentCard": false },
  "skills": [
    {
      "id": "reconcile-payment-batch",
      "name": "Reconcile a payment batch",
      "tags": ["finance", "invoices", "reconciliation"]
    }
  ]
}
```

重点是 `supportedInterfaces`。v1.0 把原先四个顶层字段（`url`、`preferredTransport`、`additionalInterfaces`、`protocolVersion`）合并成这一个**有序**数组：第一项就是首选入口，每项必须同时给出 `url`、`protocolBinding`、`protocolVersion`，缺一个是错误而不是回退。`protocolBinding` 的核心取值是 `JSONRPC`、`GRPC`、`HTTP+JSON`。

客户端读完这张卡就能定三件事：这个 agent 能不能干我要的活（看 `skills`）、请求该怎么发（看 `supportedInterfaces`）、要带什么凭证（看安全声明）。

## 三、固定路径只覆盖了一半

well-known 解决的是「已经知道域名之后」。域名本身从哪来，A2A 另给了两条路：

- **registry（目录服务）**。中介维护一批 card，客户端按 skill、tag、provider 过滤。适合企业内网和公开市场，代价是得自己运营，而且规范明确没有规定 registry 的 API 长什么样。
- **直接配置**。域名写死在配置文件或环境变量里。关系固定的场景够用，缺点是 card 一变就要改客户端。

所以真实拓扑通常是两段拼起来：registry 或配置负责给出域名，well-known 负责给出细节。

![两段发现](/assets/images/ai-agent/a2a-well-known-agent-card/body-split-v1.png)

## 四、两个会踩的坑

**路径改过名**。v0.3 之前是 `/.well-known/agent.json`，老 SDK 还在按旧路径找。稳妥做法是旧路径 308 跳到新路径，而不是二选一。

**缓存别漏**。card 变化频率很低，服务端应该带 `Cache-Control` 和 `ETag`（可以由 card 的 `version` 或内容哈希生成），客户端过期后用 `If-None-Match` 做条件请求，而不是每次重新下载整张卡。

还有一个安全前提：card 会暴露内部 URL 和敏感 skill 描述。真要放敏感信息就别放在公开的 well-known 上，走 authenticated extended card，并且用带外动态凭证，不要把静态 secret 写进卡里。

## 结论

A2A 的 agent 发现不是一套协议，而是一个约定：路径钉死在 `/.well-known/agent-card.json`，发现就退化成一次可缓存的 GET。真正需要设计的在它上游——域名怎么来，以及这张卡该给谁看。
