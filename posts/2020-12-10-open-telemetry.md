---
title: OpenTelemetry的一些随笔(1)
description: OpenTelemetry
date: 2020-12-10
tags:
	- golang
	- opentelemetry
	- cloudnative
layout: layouts/post.njk
---

中文网站上面关于 OpenTelemetry 的介绍非常之少，仅有的也只是稍微介绍一下发展历史，并没有深入到它的架构、里面所用到的一些基础概念。

当然在开始之前，还是需要稍微提及一下 OpenTelemetry 的概念。

## Observability

最近呈冉冉向上之势的 CNCF，在前段时间出了个[Trail Map](https://github.com/cncf/trailmap)，上面洋洋洒洒写了十大项。大体意思就是呢，你要搞 Cloud Native 是不是，那你对着这张图看看你到底有没有脸说。

这十大项是

- Containerized 容器化 - 现在的语境下面基本上就是 docker 化
- CI/CD 持续集成/持续交付 - 这个应该也不用解释了
- Orchestration & Application Definition 容器编排 - 当前语境下就是有没有上 k8s
- Observability & Analysis 可观察 & 分析 - 这就是本文要讲的部分
- Service Proxy, Discovery & Mesh 服务发现/代理/网络 - 也是为了治理 micro service 而引入的 buzzwords
- Networking, Policy & Security 网络政策 - 在云原生下的网络管理
- Distributed Database & Storage 分布式数据库 & 存储 - 这个就不用说了
- Streaming & Messaging 消息化 - 很多公司以为就是用上 kafka 或者任何 mq 就消息化了，实际上关于这一块有非常多可以讨论的方向。很多大厂的人都对此一窍不通，毕竟异步架构比同步架构复杂性增加无数倍。
- Software Distribution 软件分发 - 这块不是很懂，大概是安全性/P2P 分发之类的，对基础设备非常看重的大厂才会用吧。

这里面有几项，基本上像点样的公司都在做，比如容器化 & CI/CD & k8s & 分布式数据库；但其它部分就基本上是深水区了，不到一定规模估计是用不上的（包括 Service Mesh，很多公司体量根本不需要用，就算用也是当成大号 nginx 来用）。

而深水区之一，就是本文要讨论的主题：observability。

## 为什么是深水区？

因为这在我看来是属于跟 CI/CD 一样的基础设施，但很少有公司从一开始就考虑。

在这个言必称 Cloud Native 的时代，张口 docker（哪怕 dockerfile 写得跟 xml 一样又臭又长），闭口 kubernetes（哪怕完全不理解 chart 只会东抄一点西抄一点），service mesh、distribution 这些 buzzword 满大街飞，observability 被提及的概率实在是太小了点。

为什么？因为实在不够高大上。

observability 说到底，只是用来看看整个系统运行状态，用于报警或者 debug 用的。整个系统需要记录的三样东西也非常没有技术含量：logs/metrics/traces。

- logs：就是日志，毫无技术含量。CNCF 项目中有 fluentd。
- metrics：就是系统在运行时的一些数据，比如 count（数数，只增不减）、gauge（也是数数，可增可减，反应一段时间内的变化）等。CNCF 项目中有 prometheus。
- traces：记录某个 request 或者某条 event 在它的生命周期内，在不同的 service 留下的痕迹。简单来说，可以理解为集中式日志，而不用在不同service log之间跳来跳去查找问题。

