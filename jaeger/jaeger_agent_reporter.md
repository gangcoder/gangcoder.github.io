<!-- ---
title: jaeger agent reporter
date: 2020-04-23 12:50:35
category: showcode, jaeger
--- -->

# Jaeger Agent Reporter 实现

Agent 数据接收服务，接收通过UDP 协议传输过来的span 数据，并且提交到Collector。


![](images/jaeger_agent_reporter.svg)

## 1. UDP 处理服务

主要数据结构：

```go
// Processor 处理不同格式的span 数据
type Processor interface {
    Serve()
}

// AgentProcessor 处理器使用processor 处理thrift 协议数据并且调用report 上报数据
type AgentProcessor interface {
    Process(iprot, oprot thrift.TProtocol) (success bool, err thrift.TException)
}

// UDP 网络连接
type TUDPTransport struct {
    conn     *net.UDPConn // UDP 连接
}

// 网路数据缓冲处理
type TBufferedServer struct {
    dataChan      chan *ReadBuf // 缓冲数据
    transport     thrift.TTransport //UDP 连接
}
```

创建默认编码处理器：

```go
// github.com/jaegertracing/jaeger/cmd/agent/app/flags.go
// InitFromViper 根据默认处理器初始化Builder 实例
func (b *Builder) InitFromViper(v *viper.Viper) *Builder {
    // 默认协议处理处理器
    for _, processor := range defaultProcessors {
        p := &ProcessorConfiguration{Model: processor.model, Protocol: processor.protocol}
        p.Server.HostPort = v.GetString(prefix + suffixServerHostPort)
        b.Processors = append(b.Processors, *p)
    }

    b.HTTPServer.HostPort = v.GetString(httpServerHostPort)
    return b
}
```

创建UDP 处理服务：

```go
// 创建处理服务
// github.com/jaegertracing/jaeger/cmd/agent/app/builder.go
func (b *Builder) getProcessors(rep reporter.Reporter, mFactory metrics.Factory, logger *zap.Logger) ([]processors.Processor, error) {
    // 创建处理器服务
    retMe := make([]processors.Processor, len(b.Processors))
    for idx, cfg := range b.Processors {
        // ...
        
        // 数据处理handler
        var handler processors.AgentProcessor
        handler = jaegerThrift.NewAgentProcessor(rep)
        
        // UDP 服务监听
        processor, err := cfg.GetThriftProcessor(metrics, protoFactory, handler, logger)
        // ...
        retMe[idx] = processor
    }
    return retMe, nil
}
```

数据处理handler，用于处理数据接收。这里会直接将数据提交到Collector。

```go
// github.com/jaegertracing/jaeger/thrift-gen/jaeger/agent.go
// 创建Thrift 处理Agent
func NewAgentProcessor(handler Agent) *AgentProcessor {
    // ...
    self6.processorMap["emitBatch"] = &agentProcessorEmitBatch{handler: handler}
    return self6
}

// handler 接口实现
func (p *AgentProcessor) Process(iprot, oprot thrift.TProtocol) (success bool, err thrift.TException) {
    // 获取处理器，进行数据上报
    if processor, ok := p.GetProcessorFunction(name); ok {
        return processor.Process(seqId, iprot, oprot)
    }
}

// 最终调用 Report 的 EmitBatch 函数发送数据
func (p *agentProcessorEmitBatch) Process(seqId int32, iprot, oprot thrift.TProtocol) (success bool, err thrift.TException) {
    // ...
    if err2 = p.handler.EmitBatch(args.Batch); err2 != nil {
        return true, err2
    }
}
```

## 2. 服务监听实现

基于Thrift 数据编码的UDP 监听。

```go
func (c *ProcessorConfiguration) GetThriftProcessor(
    mFactory metrics.Factory,
    factory thrift.TProtocolFactory,
    handler processors.AgentProcessor,
    logger *zap.Logger,
) (processors.Processor, error) {
    // 创建服务监听
    server, err := c.Server.getUDPServer(mFactory)
    
    // 数据处理逻辑
    return processors.NewThriftProcessor(server, c.Workers, mFactory, factory, handler, logger)
}
```

### 2.1 UDP 监听

```go
// getUDPServer 创建一个自带缓冲层的Server
func (c *ServerConfiguration) getUDPServer(mFactory metrics.Factory) (servers.Server, error) {
    // 监听端口
    transport, err := thriftudp.NewTUDPServerTransport(c.HostPort)
    
    // Thrift 数据缓冲
    return servers.NewTBufferedServer(transport, c.QueueSize, c.MaxPacketSize, mFactory)
}
```

UDP 网络监听：

```go
// 监听端口
func NewTUDPServerTransport(hostPort string) (*TUDPTransport, error) {
    // 监听端口
    addr, err := net.ResolveUDPAddr("udp", hostPort)
    conn, err := net.ListenUDP(addr.Network(), addr)
    // ...
    return &TUDPTransport{addr: conn.LocalAddr(), conn: conn}, nil
}

// Read 读取UDP 网络数据包
func (p *TUDPTransport) Read(buf []byte) (int, error) {
    n, err := p.conn.Read(buf)
    return n, thrift.NewTTransportExceptionFromError(err)
}
```

读取 UDP 网络上的数据：

```go
// github.com/jaegertracing/jaeger/cmd/agent/app/builder.go
// 开启服务 server, err := c.Server.getUDPServer(mFactory)
// NewTBufferedServer 创建buffer 服务
func NewTBufferedServer(
    transport thrift.TTransport,
    maxQueueSize int,
    maxPacketSize int,
    mFactory metrics.Factory,
) (*TBufferedServer, error) {
    dataChan := make(chan *ReadBuf, maxQueueSize)
    // ...
    res := &TBufferedServer{dataChan: dataChan,
        transport:     transport,
    }
    // ...
    return res, nil
}

// Serve 网络监听服务
func (s *TBufferedServer) Serve() {
    for s.IsServing() {
        // 读取缓冲数据
        readBuf := s.readBufPool.Get().(*ReadBuf)
        n, err := s.transport.Read(readBuf.bytes)
        if err == nil {
            select {
            case s.dataChan <- readBuf:
            }
        }
    }
}

// DataChan 获取数据缓冲channel
func (s *TBufferedServer) DataChan() chan *ReadBuf {
    return s.dataChan
}
```

### 2.2 UDP 请求处理逻辑

处理UDP 请求数据。

```go
// github.com/jaegertracing/jaeger/cmd/agent/app/processors/thrift_processor.go
// NewThriftProcessor 创建基于Thrift 协议的处理器
func NewThriftProcessor(
    server servers.Server,
    numProcessors int,
    mFactory metrics.Factory,
    factory thrift.TProtocolFactory,
    handler AgentProcessor,
    logger *zap.Logger,
) (*ThriftProcessor, error) {
    // ...
    res := &ThriftProcessor{
        server:        server,
        handler:       handler,
    }
    
    // 多goroutine 并发处理
    for i := 0; i < res.numProcessors; i++ {
        go func() {
            // 读取UDP 网络数据并处理
            res.processBuffer()
        }()
    }
    return res, nil
}

// processBuffer 从channel 获取数据，然后调用handel 处理
func (s *ThriftProcessor) processBuffer() {
    // 从channel 获取数据
    for readBuf := range s.server.DataChan() {
        payload := readBuf.GetBytes()
        // 格式化数据
        protocol.Transport().Write(payload)
        
        // 将数据发送到Collector
        ok, err := s.handler.Process(protocol, protocol)
        
        // ...
    }
}

// Serve 开启服务接收数据
func (s *ThriftProcessor) Serve() {
    s.server.Serve()
}
```


## 参考资料

- github.com/jaegertracing/jaeger/cmd/agent/app/servers/tbuffered_server.go
