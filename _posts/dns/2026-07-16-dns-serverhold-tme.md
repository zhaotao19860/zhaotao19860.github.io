---
title: "不是服务器宕机：Telegram 短链接失效背后的 serverHold"
date: 2026-07-16 20:06:38 +0800
category: DNS
tags: [DNS, EPP, Registry, Availability]
excerpt: "t.me 一度从正常 DNS 委派链路中消失，暴露了命名入口高可用的盲区"
---
![cover](/assets/images/dns/dns-serverhold-tme/cover-v1.png)

一个只有四个字符的域名失效，会影响多大？对 Telegram 来说，`t.me` 不只是一张网页。用户名、频道、群组、机器人和大量站外分享链接，都通过 `t.me/*` 进入应用。入口看似分散，最终却收敛到同一个注册对象。

7 月 13 日，用户开始发现 `t.me` 无法正常解析。公开注册信息一度显示该域名处于 `serverHold`。随后域名恢复，但这次故障留下了一个比“DNS 配错了”更重要的问题：服务器、机房和权威 DNS 都可以有冗余，域名注册状态本身仍可能是单点。

## 不是服务器宕机，而是域名没有正常激活

ICANN 将 `serverHold` 列为由注册局设置的 EPP 状态，并说明处于该状态的域名不会在 DNS 中正常激活。这里的 `server` 指设置主体是 registry，而不是 Telegram 的业务服务器；`Hold` 也不等于域名已经删除。注册数据库里仍可能保留注册人、到期日和名称服务器，父区却不再正常发布委派。

这和后端宕机的表现不同。服务器故障时，DNS 往往还能返回 IP，只是连接或应用请求失败；`serverHold` 则发生在更上游，递归解析器可能根本无法沿父区找到 `t.me` 的权威名称服务器。

![body-ai](/assets/images/dns/dns-serverhold-tme/body-ai-v1.png)

同样会直接影响解析的还有 `clientHold`，但设置主体通常是注册商。`transferProhibited`、`updateProhibited` 等锁定状态一般不会让网站下线，反而常用于防盗转。监控系统不能看到一个 `prohibited` 就报警，真正需要最高优先级处理的是 `Hold`、`inactive`、`redemptionPeriod` 和 `pendingDelete` 等会影响解析或域名存续的状态。

## serverHold 在哪一层切断 DNS

一次正常解析大致经过四步：根区指向 `.ME` 顶级域，`.ME` 返回 `t.me` 的 NS 委派，递归解析器再向这些 NS 查询 A/AAAA 记录，客户端最后连接得到的地址。

`serverHold` 的关键影响在第二步。注册局根据注册数据库生成顶级域区数据时，不再把该域名作为正常委派发布。Telegram 自己的权威 DNS 中即使仍保存 A 记录，普通解析器也不会绕过父区，凭记忆猜测原来的权威服务器。

![body-fact](/assets/images/dns/dns-serverhold-tme/body-fact-v1.png)

缓存会让故障和恢复都呈现渐进过程。已经缓存 NS 或 A 记录的网络可能暂时还能访问；TTL 到期后，不同地区陆续失败。解除 `serverHold` 后，注册局重新发布委派，旧的否定缓存和委派缓存仍可能让全球恢复时间不完全一致。

因此，多机房、多云或多个权威 DNS 服务商都不能直接绕过注册局级暂停。它们保护的是域名之下的执行路径，而 `serverHold` 控制的是进入这些路径之前的命名入口。

## 为什么常规故障排查容易走错方向

域名无法访问时，值班人员通常先检查应用日志、负载均衡、证书、权威 DNS 和云厂商状态。这套顺序能处理多数故障，却可能在 `serverHold` 场景里浪费关键时间：后端节点仍然健康，原权威 DNS 也可能继续保存完整记录，真正缺失的是父区委派。只盯着业务控制台，很容易得出“所有组件都正常，但用户就是打不开”的矛盾结论。

更有效的判断方式是从外向内分层取证。先用多个递归解析器确认现象，再直接询问顶级域权威服务器是否仍返回 NS 委派；随后查询 RDAP 或 WHOIS 的 EPP 状态，最后才检查子区记录和应用服务。若父区没有委派，而已知权威服务器仍能回答 A/AAAA 记录，故障边界就已收敛到注册局或注册商控制面。

恢复判断也不能只看一次 `dig` 成功。应同时记录注册状态、父区委派、不同网络的递归结果和 HTTPS 可用性，并区分正缓存与否定缓存的 TTL。否则，某个监测点已经恢复可能被误报为全球恢复，另一个仍持有否定缓存的地区则继续失败。

这套排查顺序的价值不只是缩短定位时间。它还能让团队在联系注册商时提供明确证据：何时观察到状态变化、哪些父区节点不再返回委派、哪些递归网络受到影响，以及业务层是否仍健康。注册局级事件需要的是跨层证据，而不是不断重启本来就正常的服务器。

## OFAC 记录能证明什么，不能证明什么

美国财政部 OFAC 7 月 13 日的官方记录将 First VPN Service 列为受制裁实体，并把 `t.me/FirstVPNService` 列为该实体的一个网站。Domain Name Wire 的后续更新援引 `.ME` 注册局说法，将暂停与 “OFAC compliance” 联系起来。

可以确认的是：制裁对象是 First VPN Service，不是 Telegram；官方记录确实包含一条 `t.me` 路径；`t.me` 随后出现注册局级 `serverHold` 并造成解析故障。

不能仅凭这些公开信息确认的是，注册局内部依据什么流程，把针对一条具体频道地址的合规判断扩展成整个域名的暂停。把事件写成“Telegram 被美国制裁”不准确；断言“美国要求关闭整个 t.me”同样超出现有证据。

这里存在治理粒度错位。DNS 和注册局管理的是域名，不理解 `/FirstVPNService` 代表哪个频道。一旦使用域名级措施，技术效果天然覆盖该域名下所有无关路径。

## 一个短域名如何变成全局入口单点

短域名的价值在于统一、易记和易传播，风险也来自同一个“统一”。当用户名、频道、机器人、登录跳转和站外分享都压在 `t.me` 上，一个注册状态变化就可能同时切断大量原本独立的业务入口。

这类故障暴露了高可用设计的分层盲区：应用层可以切流，服务器层可以扩容，权威 DNS 可以更换供应商；但域名注册层被暂停时，下方冗余都还在，用户却无法到达它们。

**我的技术判断是：高可用不能只覆盖计算、网络和权威 DNS，还必须把注册商、注册局状态和备用命名入口纳入控制面。** 域名不是品牌部门的一项静态资产，而是生产系统的上游依赖。

## 给平台团队的五项准备

第一，备用域名必须能独立解析并承载关键页面，不能只是 301 跳回主域名。第二，在客户端、SDK 或桌面应用中预置经过验证的备用入口，并提前演练切换。

第三，分离品牌主站、短链接、登录认证、API 发现和运维控制面，避免一个短域名同时切断业务和恢复渠道。第四，除 HTTP 和 DNS 可用性外，持续监控 RDAP/WHOIS 中的 `serverHold`、`clientHold`、名称服务器变更、转移和续费状态。

第五，为平台、注册商和注册局建立紧急升级路径。涉及具体内容时优先推动内容级或账户级处置；如果必须采取域名级措施，应同步评估无关用户、公告渠道和恢复步骤。

截至 7 月 16 日核验时，`t.me` 已恢复正常解析，注册信息也不再显示 `serverHold`。但“恢复了 IP”只说明用户路径重新可用，不能解释暂停的完整决策链，也不意味着命名入口风险已经消失。

服务器冗余解决的是“后端还能不能工作”；域名韧性解决的是“用户还能不能找到入口”。两者缺一不可。

## 参考来源

- [OFAC：Cyber-related Designations，2026-07-13](https://ofac.treasury.gov/recent-actions/20260713)
- [Identity Digital：t.me RDAP 记录](https://rdap.identitydigital.services/rdap/domain/t.me)
- [Domain Name Wire：Telegram's t.me domain suspended, leading to outages](https://domainnamewire.com/2026/07/13/telegrams-t-me-domain-suspended-leading-to-outages/)
- [ICANN：EPP Status Codes](https://www.icann.org/resources/pages/epp-status-codes-2014-06-16-en)
- [IETF RFC 5731：EPP Domain Name Mapping](https://datatracker.ietf.org/doc/html/rfc5731)
