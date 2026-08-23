---
layout: page
lang: en
title: English
description: ZhaoTao (Tom) — ten years building authoritative and recursive DNS data planes, four production systems, three of them on DPDK. Now working on L4 gateways and hardware offload.
---

## ZhaoTao / 赵涛

Software engineer at Baidu, cloud infrastructure. I build and operate **L4
gateways (network load balancers)** and **DNS**.

For ten years I have been doing variations on one problem: rebuilding DNS from
protocol handling down to data-plane performance. Four production systems — a
carrier recursive cache, an internet-facing authoritative service, an in-VPC
authoritative and forwarding cache, and an internet-facing authoritative service
together with its control system. The last three run on self-built DPDK data
planes, mostly in C. Since 2024 I have moved to L4 gateways and hardware
offload.

Most of this blog is in Chinese. These pages are the English subset — the ones
where the engineering travels better than the language.

- [About](/en/about/) — background, systems I have built, and how to reach me
- [中文主站](/) — the full blog: DNS, DPDK data planes, AI gateways

---

## Writing

{% assign en_posts = site.en | where_exp: 'p', 'p.list != false' | sort: 'date' | reverse %}
{% if en_posts.size > 0 %}
{% for p in en_posts %}
### [{{ p.title }}]({{ p.url | relative_url }})

{% if p.date %}*{{ p.date | date: '%B %-d, %Y' }}*{% endif %}

{{ p.description }}
{% endfor %}
{% else %}
Nothing here yet.
{% endif %}

---

## Elsewhere

- GitHub — [zhaotao19860](https://github.com/zhaotao19860)
- Email — `zhaotao19860@qq.com`
