---
title: "Making Authoritative DNS Fast Is the Easy Part"
date: 2026-08-23 10:00:00 +0800
lang: en
description: Kernel bypass gets an authoritative DNS server to millions of queries per second without much heroism. What it actually costs you is everything the kernel used to do for free — and the failure mode that follows does not page you.
---

When people hear that an authoritative DNS server does tens of millions of
queries per second on a self-built DPDK data plane, they ask about the data
plane. How many cores, which NIC, what the packet path looks like.

That is the part I worry about least. Authoritative DNS is unusually friendly to
kernel bypass, and once you commit to it the throughput number mostly falls out
of the problem shape. The work that actually consumed years — and the outages
that actually happened — were about configuration state, not packets.

I have built four DNS systems over ten years: a carrier recursive cache, an
internet-facing authoritative service, an in-VPC authoritative and forwarding
cache, and an internet-facing authoritative service with its control system. The
last three run on DPDK. This is what I would tell someone starting the fourth
one.

## Why authoritative DNS is an easy case for kernel bypass

Most of what makes high-performance networking hard is absent here.

A query is **stateless**. There is no connection, no handshake, no sequence
space, no reassembly in the common path. A packet arrives, you parse a name, you
look it up, you write a response. Nothing about request *n* constrains request
*n+1*.

The packets are **small and uniform**. Queries are typically well under 100
bytes; responses usually fit in a single MTU. You are not doing segmentation, and
you are not managing wildly varying buffer sizes. The whole workload lives in the
regime where per-packet overhead dominates — which is exactly the regime where
removing the kernel's per-packet cost pays off most.

The dataset is **read-mostly**. Zone data changes on human timescales, queries
arrive on microsecond timescales. That ratio means you can pick lookup structures
that are aggressively optimized for reads and simply rebuild them on change,
rather than paying for concurrent mutation on the hot path.

And the protocol is **overwhelmingly UDP**. Roughly 99% of authoritative traffic
in the systems I have run is UDP. TCP exists for large responses and zone
transfers, but it is a rounding error by volume — which means you do not need a
userspace TCP stack on the critical path. You need TCP to *work*, not to be fast.
On earlier systems I handled it by routing the TCP tail through KNI into the
kernel stack and leaving the fast path alone. ([Chinese writeup of that
design](/dns/2021/05/08/dns-dpdk-kni-tcp/).)

Put those four properties together and the picture is clear: this is close to
the ideal workload for bypassing the kernel. Poll mode, no syscalls per packet,
no copies, cores pinned, memory on the right NUMA node, hugepages so the TLB
stops mattering. The throughput number gets large because the problem allowed it,
not because the implementation was clever.

## What you actually pay for it

Here is the part that does not appear in benchmarks. When you bypass the kernel,
you also bypass everything the kernel was doing for you that you never had to
think about.

There is no `netstat`, no `ss`, no per-socket counters. There is no `tcpdump` on
the fast path — if you want packet capture you build it, and it has to be
something you can turn on in production without dropping traffic. There is no
`iptables`, so rate limiting and filtering become your code. There is no signal
handler that reloads a config file, because there is no config file being read by
a process the OS understands.

Every one of those absences turns into a component you own, and every component
you own is a thing that can be inconsistent with the others.

The one that matters most: **a fast data plane has to keep its answers in local
memory.** You cannot consult a database per query at microsecond latency. So the
authoritative data that a node serves is, by construction, a *replica* — and now
you are running a distributed system whose correctness property is "every replica
serves what the operator asked for."

That is the real system. The packet path is a subroutine inside it.

## Configuration rollout becomes the long pole

Once answers live in node-local memory, the interesting latency is not
query-to-response. It is *change-accepted* to *change-in-effect*. That number is
what a customer experiences when they update a record, and it is the number that
determines whether a failover actually fails over.

On the Baidu Cloud MDNS side I built rollout on Redis plus MySQL and brought that
path down to **seconds**. The split is deliberate:

**MySQL owns the truth.** Configuration changes need durability, transactions,
and an ordering you can reason about after the fact. When you are reconstructing
an incident, the question is always "what was the intended state at 14:32," and
you need one place that can answer it without ambiguity.

**Redis owns the fan-out.** Getting a change to every node quickly is a different
problem from storing it correctly, and solving both in one system means either
your database is on the hot path of every node's refresh, or your cache is your
source of truth. Neither ends well. Redis carries the notification and the
current snapshot so nodes converge fast; MySQL remains the thing you replay from.

The general principle: separate *durability* from *distribution*, and never let
the fast path's convenience become the authority for what is correct.

## The failure mode that does not page you

Here is what I did not expect going in, and what I would now build for from day
one.

In a multi-node authoritative cluster there is a wide gap between "the
configuration was pushed" and "every node is answering correctly." What falls
into that gap is **silent inconsistency**. One node missed an update, or applied
it partially, or is serving a stale generation of a zone. There is no error log.
Nothing crashed. Health checks pass, because the node is up and fast — it is just
wrong.

This is the shape of failure that conventional monitoring is structurally blind
to. Liveness checks ask "are you answering?" Latency checks ask "are you
answering quickly?" Neither asks "are you answering *correctly*?" A node can
score perfectly on both while returning an IP address that was decommissioned
last week.

And the discovery path, absent something better, is a support ticket. Some
fraction of users — the fraction that happens to hash to the bad node — sees the
wrong answer, and it takes them a while to conclude the problem is you rather
than them. That is how you get a **days-long** time to detection for a
correctness bug in a system with microsecond latency.

The fix is not subtle, it is just work: actively query every node and compare the
responses. I built that consistency monitoring, and it took time-to-detection
from **days down to minutes**.

The design question worth thinking about carefully is what you compare
*against*. Comparing nodes to each other catches divergence, which is the common
case — but it passes cleanly when every node is uniformly wrong, which is exactly
what a bad push produces. Comparing each node against the control plane's
intended state catches that too, at the cost of needing the intended state
expressed in a form you can turn into expected answers. The two checks fail in
different directions, and knowing which one you have is knowing which outages you
will still be finding out about from customers.

This is invisible infrastructure. On a normal day it produces nothing. Its entire
value is realized during incidents, which makes it perpetually hard to justify
and perpetually the thing you wish you had built sooner.

## What I would carry forward

Three things, if I were starting the next one.

**Instrument the configuration path as seriously as the query path.** Everyone
graphs QPS and p99 latency. Far fewer graph the distribution of
change-to-in-effect latency per node, which is where the user-visible failures
actually come from.

**Make the correctness oracle external to the thing it checks.** A system cannot
validate itself using the same state that might be corrupt. If the checker reads
from the data plane's cache, it will confirm whatever the data plane believes.

**Treat performance as a constraint that generates obligations, not as an
achievement.** Choosing kernel bypass is choosing to reimplement observability,
filtering, and configuration propagation. That is a fine trade for a workload
like authoritative DNS. It is only a bad trade when you make it without budgeting
for the second half.

The throughput was never the interesting part. It was the entry fee.
