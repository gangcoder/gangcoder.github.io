<!-- ---
title: jaeger guide
date: 2019-03-13 17:03:37
category: architecture, jaeger
--- -->

# Jaeger 指南

Jaeger 是一个分布式链路追踪系统。由 Uber 开源，他可以用来监控分布式微服务的调用链路，兼容 OpenTracing API。

功能包括:

- 分布式链路追踪
- 服务依赖分析
- 性能和延迟优化

## 链路追踪概述

存在这样一种场景，当我们进行微服务拆分后，一个请求将会经过多个服务处理之后再返回，这时，如果在请求的链路上某个服务出现故障时，排查故障将会比较困难．
我们可能需要将请求经过的服务，挨个查看日志进行分析，当服务有几十上百个实例时，这无疑是可怕的．因此为了解决这种问题，调用链追踪应运而生．

将请求经过的服务进行编号，统一收集起来，形成逻辑上的链路，这样，我们就可以看到请求经过了哪些服务，从而形成服务依赖的拓扑。

总链路由每段链路组成，每段链路均代表经过的服务，耗时可用于分析系统瓶颈，当某个请求返回较慢时，可以通过排查某一段链路的耗时情况，从而分析是哪个服务出现延时较高，在到具体的服务中分析具体的问题。

## 架构结构

![](images/jaeger_structure.svg)

Jaeger 组成部分：

- Jaeger Client - 为不同语言实现了符合 OpenTracing 标准的 SDK。应用程序通过 API 写入数据，client library 把 trace 信息按照应用程序指定的采样策略传递给 jaeger-agent。
- Agent - 它是一个监听在 UDP 端口上接收 span 数据的网络守护进程，它会将数据批量发送给 collector。它被设计成一个基础组件，部署到所有的宿主机上。Agent 将 client library 和 collector 解耦，为 client library 屏蔽了路由和发现 collector 的细节。
- Collector - 接收 jaeger-agent 发送来的数据，然后将数据写入后端存储。Collector 被设计成无状态的组件，因此您可以同时运行任意数量的 jaeger-collector。
- Data Store - 后端存储被设计成一个可插拔的组件，支持将数据写入 cassandra、elastic search。
- Query - 接收查询请求，然后从后端存储系统中检索 trace 并通过 UI 进行展示。Query 是无状态的，您可以启动多个实例，把它们部署在 nginx 这样的负载均衡器后面。
- ingester 消费队列消费者


## 参考资料

- https://zhuanlan.zhihu.com/p/34318538