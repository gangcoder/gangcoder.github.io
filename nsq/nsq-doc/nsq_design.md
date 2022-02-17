---
title: nsq 设计
date: 2018-08-30 10:48:59
category: language, go, nsq, doc
---

## nsq 设计

NSQ 是继承于 simplequeue(部分的 simplequeue):

• 基于分布式，提升高可用性并且消除单点故障
• 保证更稳定的消息可靠传递
• 通过持久化一些消息到硬盘上，来限制单个进程的内存占用
• 极大简化了生产者和消费者的配置要求


### 消息模型

nsqd 设计成可以同时处理多个数据流，数据流被称为“话题” `topic`，话题有1 个或多个“通道” `channels`，每个通道都接收话题中所有消息的副本，同时每个通道对应到多个消费者。

话题和通道不用预先配置，话题在第一次发布消息或第一次订阅时创建，通道也会在第一次订阅时创建。

话题和通道的所有数据相互独立，这样能防止某个消费者处理缓慢造成对其他通道的积压，同样适用于话题级别。一个通道一般会有多个客户端连接，假设所有已连接的客户端处于准备接收消息的状态，消息将平均且随机地传递到某个客户端。

简单来说，“话题”有1 个或多个“通道”，每个通道都接收到一个话题中所有消息的副本，消息从话题->通道是多路传送的。当有多个消费者时，每个消费者只能收到“通道” 中一部分的消息，消息从“通道” -> 消费者是平均分布的。


### nsqlookupd 发现服务

nsqlookupd 提供了一个发现服务，消费者可以查找到订阅话题的 nsqd 地址。nsqlookupd 将消费者与nsqd 解耦，这样它们只需要连接到nsqlookupd，可以极大降低复杂度。

层面是线上，每个nsqd 会建立与nsqlookupd 的TCP 连接，并且定期推动其状态，这样nsqlookupd 就能将nsqd 的地址通知消费者。nsqlookupd 提供HTTP 接口供消费者查询。

为话题引入新的消费者时，消费者可以通过nsqlookup 找到nsqd 的地址，大大降低了维护成本。


### 消除单点故障

nsq 被设计为分布式方式工作，所有的消费者连接到NSQD 实例，直接消费消息，没有中间层，由于每个有“通道”都有足够的消费者，可以保证所有的消息最终能被消费，从而达到消除单点故障的目的。

nsqlookupd 的高可用性是通过运行多个实例来实现，nsqlookupd 实例之间不相互通信，但是消费者可以查询多个实例，并且把查询结果合并，从而保证数据一致性和可靠性。


### 消息传递保证

nsq 保证一个消息至少投递一次，由客户端负责实现幂等操作。假设客户端成功连接并订阅一个话题，工作原理如下：

1. 客户表示他们已经准备好接收消息
2. NSQ 发送一条消息，并暂时将数据存储在本地（在 re-queue 或 timeout）
4. 如果客户端没有回复, NSQ 会在设定的时间超时，自动重新将消息入队并且再次推送

This ensures that the only edge case that would result in message loss is an unclean shutdown of an nsqd process. In that case, any messages that were in memory (or any buffered writes not flushed to disk) would be lost.

消息丢失的极端情况是nsqd 进程意外终止，这种情况下所以内存和正在写入磁盘的消息都会丢失。如果要避免消息丢失，可以部署两套nsqd 集群，将一份消息同时写入两个集群，并且消费者同时订阅这两个集群，这样每条消息会接收到次，因为客户端具有幂等性，所以不会对业务数据造成影响。


### 效率

所有的消息数据包括像尝试次数、时间截等元数据都被保持在nsqd 中，避免了元数据从服务器到客户端之间的传输。

对于数据的协议，nsq 通过推送数据到客户端而不是等待客户端拉数据，最大限度地提高性能和吞吐量的。这个概念，我们称之为 RDY 状态，它的实现是客户端流量控制的一种形式。

当客户端连接到nsqd 并且订阅到一个通道时，它处于RDY 为0 状态，此时没有信息被发送到客户端；当客户端准备好接收消息时，更新它的RDY 状态到它准备处理的消息数量（比如 100），无需任何额外的指令，当100 条消息可用时，将被传递到客户端。


## 内核

### 话题和通道

话题（topic）和通道（channel）是NSQ 的基本概念。Go 语言中的通道（channel）（为消除歧义简称为“go-chan”）是实现队列的一种方式，对于NSQ 话题（topic）和通道（channel），内部实现是一个带有缓冲的 go-chan Message 指针。缓冲区的大小通过 `--mem-queue-size` 参数配置。

前面了解了消费数据，发布消息到话题（topic）的行为涉及到：

1. 消息结构的初始化（和消息体的内存分配）
2. 获取`话题（topic）` 的读-锁；
3. 判断读-锁是否能发布；
4. 将消息发布到带缓冲的 go-chan

从一个话题中的通道获取消息不能依赖于经典的`go-chan` 方式，因为多个goroutines 在一个 go-chan 上接收的消息是均匀分配，最终的实现方式是复制消息到每一个通道（goroutine）。

每个话题由3 个主要的goroutines 维护:

1. `router` ，负责从 `incoming go-chan` 读取最近发布的消息，并把消息保存到队列中（内存或硬盘）。
2. `messagePump` ，是负责复制和推送消息到通道。
3. `DiskQueue IO` 负责将消息写入磁盘和从磁盘读取消息。


### Backend / DiskQueue

NSQ 通过DiskQueue 将溢出的消息写入到磁盘上，达到限定内存中消息数的目的。

具体实现如下：

```go
for msg := range c.incomingMsgChan {
    select {
    case c.memoryMsgChan <- msg:
    default: //default 语句只在 memoryMsgChan 已满的情况下执行。
        err := WriteMessageToBackend(&msgBuf, msg, c.backend)
            if err != nil {
            // ... handle errors ...
            }
    }
}
```

NSQ 还有临时通道的概念，临时的通道将丢弃溢出的消息（而不是写入到磁盘），在没有客户端订阅时消失。


### 降低 GC 的压力

在有垃圾回收机制的环境中，你可能会关注到吞吐量量（做无用功），延迟（响应），并驻留集大小（footprint）。

Go 的GC 机制在不断改进，但是需要注意的是：你创建的垃圾越少，收集的时间越少

在分析垃圾生成方面，Go toolchain 提供了工具：

1. 使用 testing package 和 go test -benchmem 来 benchmark 热点代码路径。它分析每个迭代分配的内存数量（和 benchmark 运行可以用 benchcmp 进行比较）。
2. 编译时使用 go build -gcflags -m，会输出逃逸分析的结果。

GC 优化点:

1. 避免 []byte 到 string 的转换
2. 重新利用创建的buffers 或 object 
3. 再知道承载元素的数量和大小时预先分配 slices(在 make 时指定容量)
4. 对各种配置项目使用一些明智的限制（例如消息大小）
5. 避免一些不必要的包装类型（例如一个多值的”go-chan” 结构体）
6. 避免在热点代码路径使用 defer (它也消耗内存)


### TCP 协议

NSQ 的 TCP 协议 protocol_spec，基于GC 优化概念，做到了极致。

该协议用含有长度前缀的帧构造，使其可以直接高效的编码和解码。由于提前知道了帧部件的确切类型与大小，我们避免了 encoding/binary的 Read() 和 Write() 包装（以及它们外部 interface 的查询与转换），而是直接调用相应的 binary.BigEndian 方法。


### HTTP

NSQ的 HTTP API 是基于Go 的 net/http 包。因为它只是 net/http ，它可以运行在所有现代编程环境。

Go 的 HTTP tool-chest 支持广泛的调试功能。net/http/pprof 包直接集成了原生的 HTTP 服务器，通过endpoints 提供CPU，堆，goroutine 和操作系统线程性能信息。

这些信息可以直接使用 go tool 进行分析：

```
$ go tool pprof http://127.0.0.1:4151/debug/pprof/profile
```


### 依赖

Go 依赖管理相对缺乏，NSQ 从直接包含相关的的依赖 packages。


### 健壮性

表现良好的系统是在分布式生产环境中，面对不断变化的网络条件或突发事件时，能保持健壮的系统。

NSQ 设计理念是快速失败，把错误当作是致命的，并且使系统能够容忍故障而且表现出一致性，同时提供了一种方式来调试发生的任何问题。


### 心跳和超时

NSQ 的 TCP 协议是面向 push 的。在建立连接并且订阅后，消费者被放置在一个RDY 状态为0 。当消费者准备好接收消息，它更新的 RDY 状态到准备接收消息的数量。NSQ 客户端库不断在幕后管理，消息控制流。

每隔一段时间，nsqd 将发送一个心跳线连接。客户端可以配置心跳之间的间隔，但 nsqd 会期待一个回应在它发送下一个心掉之前。

组合应用级别的心跳和 RDY 状态，避免头阻塞现象。

### 管理 Goroutines

goroutine 非常容易启动，但是很难协调他们的清理工作，避免死锁也极具挑战性。孤立的 goroutine 会导致内存泄漏，内存泄露又不利于长期稳定的运行守护进程

sync 包提供了 sync.WaitGroup, 用来统计有多少个 goroutine 是活跃的，并且会一直等待直到它们退出。

有一个简单的方式，在多个 child goroutine 中触发一个事件。提供一个 go-chane，当你准备好时关闭它。所有在那个 go-chan 上挂起的 go-chan 都将会被激活，而不用向每个 goroutine 中发送一个单独的信号


### 日志

日志可以记录goroutine 进入和退出，这使得它相当容易识别造成死锁或泄漏的问题的原因。

该日志极为详细，但不是所有的日志都很详细，nsqd 倾向于发生故障时在日志中提供更多的信息。

## 参考资料

- [internals](https://nsq.io/overview/internals.html)
- [design](https://nsq.io/overview/design.html)

