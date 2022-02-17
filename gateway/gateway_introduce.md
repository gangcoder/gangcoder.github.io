<!-- ---
title: 网关介绍
date: 2020-07-06 08:52:38
category: showcode, gateway
--- -->

# 网关介绍

## 1. API 网关

API 网关是一个服务器，是系统的单入口点。API 网关封装了内部系统架构，并针对每个客户端提供一个定制 API，常用于认证、监控、负载均衡、缓存和静态响应处理。

### 1.1 网关优点

网关封装了应用程序的内部结构。客户端只需要与网关通信，而不必调用特定的服务。API 网关为每种类型的客户端提供了特定的 API，减少了客户端与应用程序之间的往返次数，简化了客户端的代码。

### 1.2 网关缺点

网关需要开发、部署和管理，可能会成为开发瓶颈。

## 2. API 网关设计

API 网关设计要点：

### 2.1 性能与可扩展性

对于大多数应用来说，API 网关的性能和可扩展性是相当重要的。需要在一个支持异步、非阻塞 I/O 平台上构建 API 网关是很有必要的。

### 2.2 响应式编程模型

API 网关通过调用多个后端服务并聚合结果来处理其他请求。对于某些请求，API 网关应该并发执行独立请求。

### 2.3 服务调用

服务调用使用进程间（inter-process）通信机制。有两种进程间通信方案。一是使用基于消息的异步机制。另一种采用同步机制，如 HTTP 和 Thrift。

### 2.4 服务发现

API 网关需要知道与其通信的每个微服务的位置（IP 地址和端口）。API 网关需要服务发现机制：[服务端发现](http://microservices.io/patterns/server-side-discovery.html)或[客户端发现](http://microservices.io/patterns/client-side-discovery.html)。

### 2.5 处理局部故障

实施 API 网关时必须解决的另一个问题是局部故障问题。当一个服务调用另一个响应缓慢或者不可用的服务时，所有分布式系统都会出现此问题。API 网关不应该无期限地等待下游服务。

## 参考资料

- 微服务：从设计到部署 http://oopsguy.com/books/microservices/2-using-an-api-gateway.html
- 谈谈微服务中的 API 网关（API Gateway）https://www.cnblogs.com/savorboard/p/api-gateway.html
- 谈谈我想要的API网关 https://www.jianshu.com/p/48ab61ba70b8

