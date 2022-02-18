<!-- ---
title: nsq 组件
date: 2018-09-07 12:47:17
category: language, go, nsq, doc
--- -->

nsq 组件

## 1. nsqd

nsqd 是一个守护进程，负责接收，入队，分发消息给客户端，它可以独立运行，不过通常是以集群部署。

nsqd 监听2 个TCP 端口，一个给客户端，另一个是HTTP API。

### 1.1 AUTH

通过`-auth-http-address=host:port` 配置项，可以配置遵从Auth HTTP 协议的授权服务器。

nsqd 将每隔一段时间里重新请求授权，nsqd 和 授权服务器间通过TLS 加密进行网络通信。


### 1.2 处理能力处理

可以设置nsqd 收集和分发信息的处理能力，通过 `--e2e-processing-latency-percentile` 标志位来配置，使用百分比技术（参见 Effective Computation of Biased Quantiles over Data Streams）来计算处理能力值。


## 2. nsqlookupd

nsqlookupd 负责管理集群拓扑信息。客户端通过查询 nsqlookupd 查询话题（topic）的生产者，并且nsqd 会广播话题（topic）和通道（channel）信息给nsqlookupd。

nsqlookupd 有两个接口：TCP 接口给nsqd 广播消息，HTTP 接口提供给客户端查询 nsqd 信息。


## 3. TCP 协议规范

nsqd 进程通过监听配置的TCP 端口来接受客户端连接。连接后，客户端发送 4 字节的标识码，标识通讯协议的版本。然后客户端可以发送 IDENTIFY 命令到服务端，协商通信的元数据和特性。

为了消费消息，客户端必须订阅 (SUB) 到一个通道（channel)。

订阅的时候，客户端的 RDY 状态为 0。意味着没有消息会被发送到客户端。当客户端已经准备好接受消息时，需要发送命令设置RDY，比如设置为 100，服务端将100 条消息推送到客户端（每次服务端都会相应的减少 RDY 的值）。

V2 版本的协议支持客户端心跳功能。每隔 30 秒（默认设置）， nsqd 将会发送一个 `_heartbeat_` 并期待返回。如果客户端响应 NOP 命令。如果 2 个 `_heartbeat_` 没有被应答， nsqd 将会强制关闭客户端连接。


## 参考资料

- [components nsqd](https://nsq.io/components/nsqd.html)
- [tcp_protocol_spec](https://nsq.io/clients/tcp_protocol_spec.html)
- [Effective Computation of Biased Quantiles over Data Streams](http://www.cs.rutgers.edu/~muthu/bquant.pdf)