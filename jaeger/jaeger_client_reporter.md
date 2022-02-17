<!-- ---
title: jaeger reporter
date: 2019-08-13 15:06:06
category: showcode, jaeger
--- -->

# Jaeger Client Reporter 实现

> jaeger client reporter 用于将客户端的span 数据上报到agent 或者collector。

reporter 逻辑流程：

1. 创建Reporter 实例
2. 创建sender 发送器，发送器可以使用http 协议，也可以使用udp 协议
3. Reporter 实现Reporter 接口，通过Reporter 接口提交的数据由 sender 发送到远程
4. report 上报器调用发送器的Append 接口将span 提交到发送器，当sender 缓冲区满了会调用 Flush 接口发送数据

![](images/jaeger_client_reporter.svg)


## 1. 创建Reporter 上报器

主要结构和需要实现的接口:

```go
// Reporter 上报器需要实现的接口
type Reporter interface {
    // Report 上报span 数据，上报的数据会异步处理
    Report(span *Span)
    // ...
}

// 远程上报器实例
type remoteReporter struct {
    // 发送器
    sender Transport
    // 异步数据channel
    queue  chan reporterQueueItem
}

// 发送器需要实现的接口
type Transport interface {
    // Append 将span 放入发送器缓存中
    Append(span *Span) (int, error)
    // Flush 将缓冲中数据提交到远程服务
    Flush() (int, error)
}
```

创建Tracer 实例时创建Reporter：

```go
// github.com/jaegertracing/jaeger-client-go/config/config.go
r, err := c.Reporter.NewReporter(c.ServiceName, tracerMetrics, opts.logger)

// github.com/jaegertracing/jaeger-client-go/config/config.go
// NewReporter instantiates a new reporter that submits spans to the collector
func (rc *ReporterConfig) NewReporter(
    serviceName string,
    metrics *jaeger.Metrics,
    logger jaeger.Logger,
) (jaeger.Reporter, error) {
    // 创建发送器
    sender, err := rc.newTransport()
    
    // 创建reporter 实例
    reporter := jaeger.NewRemoteReporter(
        sender,
        jaeger.ReporterOptions.QueueSize(rc.QueueSize),
        jaeger.ReporterOptions.BufferFlushInterval(rc.BufferFlushInterval),
        jaeger.ReporterOptions.Logger(logger),
        jaeger.ReporterOptions.Metrics(metrics))
    
    // ...
    return reporter, err
}
```


### 1.1 创建发送器

如果配置的是 Collector 地址，走http 协议传送数据；否则走 udp 协议将数据发送给本地 jaeger agent。

发送器需要实现 `Transport` 接口。

```go
// github.com/jaegertracing/jaeger-client-go/config/config.go
func (rc *ReporterConfig) newTransport() (jaeger.Transport, error) {
    switch {
    // ...
    case rc.CollectorEndpoint != "":
        return transport.NewHTTPTransport(rc.CollectorEndpoint, transport.HTTPBatchSize(1)), nil
    default:
        return jaeger.NewUDPTransport(rc.LocalAgentHostPort, 0)
    }
}
```


### 1.2 创建远程上报器实例

```go
// github.com/jaegertracing/jaeger-client-go/reporter.go
// NewRemoteReporter 将数据上报到远程agent 或者collector
func NewRemoteReporter(sender Transport, opts ...ReporterOption) Reporter {
    // ...
    reporter := &remoteReporter{
        reporterOptions: options,
        sender:          sender,
        queue:           make(chan reporterQueueItem, options.queueSize),
    }
    go reporter.processQueue()
    return reporter
}
```

远程上报器异步处理上报的span 数据。

```go
// github.com/jaegertracing/jaeger-client-go/reporter.go
// processQueue
// 1. 定时发送sender 发送器中的缓冲span 数据
// 2. 将上报器队列 channel 中的span 添加到发送器缓冲中
func (r *remoteReporter) processQueue() {
    // ...
    timer := time.NewTicker(r.bufferFlushInterval)
    for {
        select {
        case <-timer.C:
            // 定时刷新缓冲处理
            flush()
        case item := <-r.queue:
            // ...
            // 异步接收上报的span 数据，添加到发送器的缓冲区中
            r.sender.Append(span);
            // ...
        }
    }
}
```


### 1.3 上报器实现Report 接口

上报器实现都需要提供 `Report` 接口，用与将span 数据发送到远端收集器。

`Report` 将数据写入 `queue` channel 中，由异步goroutine 处理。


```go
// 远程上报器 Report 接口实现
// 将span 存储 queue channal 中，由其他goroutine 异步处理
func (r *remoteReporter) Report(span *Span) {
    select {
    // Need to retain the span otherwise it will be released
    case r.queue <- reporterQueueItem{itemType: reporterQueueItemSpan, span: span.Retain()}:
        atomic.AddInt64(&r.queueLength, 1)
    default:
        r.metrics.ReporterDropped.Inc(1)
    }
}
```

## 2. udp 发送器实现

默认走 udp 协议将数据发送给本地 jaeger agent。

```go
// github.com/jaegertracing/jaeger-client-go/transport_udp.go
// NewUDPTransport 创建基于UDP 协议的上报器，提交span 信息到 jaeger agent
func NewUDPTransport(hostPort string, maxPacketSize int) (Transport, error) {
    // ...
    client, err := utils.NewAgentClientUDP(hostPort, maxPacketSize)
    if err != nil {
        return nil, err
    }

    sender := &udpSender{
        client:         client,
        maxSpanBytes:   maxPacketSize - emitBatchOverhead,
        thriftBuffer:   thriftBuffer,
        thriftProtocol: thriftProtocol}
    return sender, nil
}
```

### 2.1 创建udp 客户端

创建到agent 的UDP 连接客户端。

```go
// github.com/jaegertracing/jaeger-client-go/utils/udp_client.go
// NewAgentClientUDP 创建到agent 的客户端
func NewAgentClientUDP(hostPort string, maxPacketSize int) (*AgentClientUDP, error) {
    // 创建到agent thrift 数据序列化实例
    client := agent.NewAgentClientFactory(thriftBuffer, protocolFactory)

    // 开启upd 连接
    destAddr, err := net.ResolveUDPAddr("udp", hostPort)
    connUDP, err := net.DialUDP(destAddr.Network(), nil, destAddr)

    // ...
    clientUDP := &AgentClientUDP{
        connUDP:       connUDP,
        client:        client,
        maxPacketSize: maxPacketSize,
        thriftBuffer:  thriftBuffer}
    return clientUDP, nil
}

// EmitBatch 实现thrift 协议接口
func (a *AgentClientUDP) EmitBatch(batch *jaeger.Batch) error {
    // 将数据转为 thrift 协议格式
    if err := a.client.EmitBatch(batch); err != nil {
        return err
    }
    
    // ...
    // 通过UDP 协议发送数据
    _, err := a.connUDP.Write(a.thriftBuffer.Bytes())
    return err
}
```

### 2.2 基于UDP 实现Transport 接口

UDP 实现Transport 接口:

```go
func (s *udpSender) Append(span *Span) (int, error) {
    // 序列化span 数据
    jSpan := BuildJaegerThrift(span)
    spanSize := s.calcSizeOfSerializedThrift(jSpan)
    // ...

    s.byteBufferSize += spanSize
    // 缓冲区数据小于缓冲区限制时，将数据放入临时slice 中
    if s.byteBufferSize <= s.maxSpanBytes {
        // 添加span 到缓冲区
        s.spanBuffer = append(s.spanBuffer, jSpan)
        // 如果数据没有达到缓冲区最大限制，添加完后后返回
        if s.byteBufferSize < s.maxSpanBytes {
            return 0, nil
        }
        
        // 添加数据后达到缓冲区限制，发送所有数据
        return s.Flush()
    }

    // ...
    return n, err
}

// 发送所有缓冲区中的数据
func (s *udpSender) Flush() (int, error) {
    n := len(s.spanBuffer)
    
    // ...
    // 发送数据
    err := s.client.EmitBatch(&j.Batch{
        Process: s.process,
        Spans:   s.spanBuffer,
        SeqNo:   &batchSeqNo,
        Stats:   s.makeStats(),
    })

    // 重置缓冲区
    s.resetBuffers()

    return n, err
}
```


## 3. http 发送器实现

使用http 发送span 数据。这里默认是发送到Collector 。

```go
// 创建http 协议的发送器
transport.NewHTTPTransport(rc.CollectorEndpoint, transport.HTTPBatchSize(1)), nil

// github.com/jaegertracing/jaeger-client-go/transport/http.go
// 用于将数据发送到 collector
func NewHTTPTransport(url string, options ...HTTPOption) *HTTPTransport {
    c := &HTTPTransport{
        url:       url,
        client:    &http.Client{Timeout: defaultHTTPTimeout},
        batchSize: 100,
        spans:     []*j.Span{},
    }

    // ...
    return c
}

// 实现Append 接口
func (c *HTTPTransport) Append(span *jaeger.Span) (int, error) {
    // ...
    // 序列化数据
    jSpan := jaeger.BuildJaegerThrift(span)
    
    // 添加数据，如果数据超过缓冲大小就发送缓冲区数据
    c.spans = append(c.spans, jSpan)
    if len(c.spans) >= c.batchSize {
        return c.Flush()
    }
    return 0, nil
}

// 实现Flush 接口
func (c *HTTPTransport) Flush() (int, error) {
    count := len(c.spans)
    // ...
    
    err := c.send(c.spans)
    c.spans = c.spans[:0]
    return count, err
}
```

发送数据：

```go
func (c *HTTPTransport) send(spans []*j.Span) error {
    batch := &j.Batch{
        Spans:   spans,
        Process: c.process,
    }

    // 序列化数据
    body, err := serializeThrift(batch)
    // ...
    req, err := http.NewRequest("POST", c.url, body)
    
    // ...
    resp, err := c.client.Do(req)
    // ...
    resp.Body.Close()
    if resp.StatusCode >= http.StatusBadRequest {
        return fmt.Errorf("error from collector: %d", resp.StatusCode)
    }
    return nil
}
```

## 参考资料

- github.com/jaegertracing/jaeger-client-go/reporter.go

