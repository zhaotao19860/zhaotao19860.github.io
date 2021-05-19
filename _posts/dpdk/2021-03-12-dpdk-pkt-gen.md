---
title: dpdk发包工具架构比较
date: 2021-05-19 19:20:00 +0800
category: DPDK
---
基于DPDK的发包工具，具有开源、软件实现、性能高等优点，可用于测试交换机、路由器、负载均衡器、防火墙、IPS、IDS等网络层设备及应用。常用工具有pktgen-dpdk,moongen,t-rex,warp17等，本文主要说明这四个软件的架构。

#### pktgen-dpdk
[pktgen-dpdk](https://git.dpdk.org/apps/pktgen-dpdk)是由intel团队成员将内核版本的pktgen移植到dpdk之上的版本，与dpdk同步版本更新最积极，目前使用的是20.11版本。
![pktgen-dpdk-architecture.png](/assets/images/dpdk/pktgen-dpdk-architecture.png)

#### moongen
[moongen](https://github.com/emmericp/MoonGen)是由个人开发和维护的版本，dpdk使用的是17.08，已经有3年没有更新了。
![moongen-architecture.png](/assets/images/dpdk/moongen-architecture.png)

#### t-rex
[t-rex](https://github.com/cisco-system-traffic-generator/trex-core)是由设备厂商思科维护的版本，社区活跃，版本更新迅速，dpdk目前已经可以兼容最新的21.02。
![trex_architecture.png](/assets/images/dpdk/trex_architecture.png)

#### warp17
[warp17](https://github.com/Juniper/warp17)是由设备厂商Juniper维护的版本，社区不是很活跃，版本较慢，dpdk目前使用的19.11。
![warp17-architecture.png](/assets/images/dpdk/warp17-architecture.png)