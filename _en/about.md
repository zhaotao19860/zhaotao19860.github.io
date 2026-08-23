---
title: About
lang: en
list: false
description: ZhaoTao (Tom), software engineer at Baidu cloud infrastructure. Ten years on DNS data planes, four production DNS systems, now L4 gateways and hardware offload.
---

## ZhaoTao / 赵涛

Software engineer at Baidu, cloud infrastructure. I build and operate L4
gateways (network load balancers) and DNS.

Ten years, one problem seen from different sides: **rebuilding DNS from protocol
implementation down to data-plane performance.** A carrier recursive cache, an
internet-facing authoritative service at JD Cloud, an in-VPC authoritative and
forwarding cache, and an internet-facing authoritative service with its control
system at Baidu Cloud. Four systems; the last three on self-built DPDK data
planes reaching **tens of millions of QPS with microsecond processing latency**,
written mostly in C. Since 2024 I have worked on L4 gateways and hardware
offload.

- **DNS — 10 years.** Authoritative (internet-facing and in-VPC), recursion and
  caching, protocol handling, rate limiting, CNAME flattening, IPv6, consistency
  monitoring.
- **L4 gateway — 2 years.** DPDK-based ingress and egress gateways, VXLAN,
  balancing-algorithm work.
- **Data plane.** DPDK, packet I/O performance, NIC bring-up.
- **Hardware offload.** Traffic steering in NPL on a TD5 switch ASIC, pushing
  part of the forwarding decision into programmable silicon.
- **What I am following now.** AI gateways and inference gateways — scheduling,
  caching, health checking, context compression. I think this is the largest
  structural change coming to the traffic ingress layer.

Languages: C (primary), Go, C++, NPL.

Contact: `zhaotao19860@qq.com` · GitHub:
[zhaotao19860](https://github.com/zhaotao19860)

---

## Some specifics

**Designing a cloud-scale internet-facing authoritative DNS from scratch (JD
Cloud ADNS, 2018–2020)**

I owned the design and implementation across three planes at once: a data plane
handling DNS protocol and security, a control plane distributing configuration,
and an analytics plane producing operational reports. The DPDK data plane reached
**tens of millions of QPS with microsecond processing latency**. The hard part
was not single-node throughput. It was keeping state consistent across the three
planes — a fast data plane has to cache configuration locally, and the moment
what the control plane pushed diverges from what the data plane serves, users get
wrong answers. That kind of failure does not page you. It arrives as a support
ticket.

**In-VPC authoritative DNS is a different problem from internet-facing
authoritative DNS (JD Cloud vpcdns, 2020–2021)**

Having built the internet-facing service, I assumed the in-VPC one was the same
system in a different deployment position. The central tension turns out to be
unrelated. Internet-facing, your adversaries are attack traffic and cache
poisoning. In-VPC, your adversaries are tenant isolation and the forwarding path:
the same name must resolve differently depending on which VPC asked, and caching
plus forwarding make that very easy to get wrong in a way that leaks across
tenants.

**Making configuration rollout and consistency checking something you can rely
on (Baidu Cloud DUDNS/MDNS, 2021–2024)**

The price of a high-performance data plane is a longer path from "configuration
accepted" to "configuration in effect." On the MDNS side I built rollout on Redis
plus MySQL and brought it down to **seconds** — MySQL for durability and
transactions, Redis for fan-out, so data-plane nodes can pull current state
quickly.

The other half is verification. In a multi-node authoritative cluster there is a
wide gap between "the configuration was pushed" and "every node is answering
correctly," and what falls into that gap is silent inconsistency: no error logs,
no alerts, just some fraction of users getting the wrong answer. I built
consistency monitoring that actively queries every node and compares responses,
which took **time-to-detection from days down to minutes**. It is invisible
infrastructure; its value only shows up during an incident.

**Pushing forwarding decisions into the switch ASIC (Baidu BGW/BNAT, 2024– )**

An L4 gateway's performance ceiling ends up being the CPU. We have been working
on hardware offload: programming a TD5 in NPL so traffic steering happens in the
switch ASIC and the CPU only handles what genuinely needs state. The trade-off is
blunt — the logic you can express in silicon is limited, and what you buy with
that constraint is an order of magnitude in forwarding capacity, paid for in
flexibility and in how hard it becomes to debug.

---

## Experience

| Period | Organization |
|---|---|
| 2021.05 – present | Baidu Times Network Technology (Beijing) · Software Engineer |
| 2018.04 – 2021.05 | Beijing Jingdong Shangke Information Technology (JD.com) · Software Engineer |
| 2015.12 – 2018.04 | Beijing Lemon Micro Fun Technology · Software Engineer |
| 2011.07 – 2015.12 | AsiaInfo Technologies (Nanjing) · Software Engineer |

M.Sc. Computer Applied Technology, Soochow University (2008–2011) · B.Sc.
Information and Computing Science, Taishan University (2004–2008)
