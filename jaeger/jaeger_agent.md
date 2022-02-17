<!-- ---
title: jaeger agent
date: 2019-07-11 10:34:12
category: src, jaeger
--- -->

# Jaeger Agent 实现

> 安装在应用实例节点上，收集实例上报的tracer 数据，然后上报到collector。

关键逻辑：

1. 创建到Collector 的客户端连接，用于上报span 数据和获取采样策略
2. 创建Agent 服务
   1. 创建UPD 处理器，接收应用传输过来的span 数据
   2. 创建HTTP 处理器，提供采样策略数据
3. 开启服务，包括http 服务和UDP 服务


![](images/jaeger_agent.svg)


## 1. 入口函数逻辑

创建本地服务接收应用程序的spans 数据，并且将数据发送到Collector。

1. 创建grpc 客户端，用于将数据发送到Collector
2. CollectorProxy 封装基于grpc 协议的上报处理逻辑
3. 创建本地服务接收应用程序的spans
4. 开启服务

主要数据结构：

```go
// ConnBuilder grpc 连接配置
type ConnBuilder struct {
    // CollectorHostPorts 存储Jaeger Collectors 的ip 和端口
    CollectorHostPorts []string `yaml:"collectorHostPorts"`
    // ...
}

// CollectorProxy 上报代理需要实现的接口
type CollectorProxy interface {
    GetReporter() reporter.Reporter
    GetManager() configmanager.ClientConfigManager
}

// ProxyBuilder 代理创建器
type ProxyBuilder struct {
    reporter *reporter.ClientMetricsReporter
    // 采样率控制器
    manager  configmanager.ClientConfigManager
    conn     *grpc.ClientConn
}

// Reporter 从Processor 接收spans，并且发送到collecotr
type Reporter interface {
    EmitBatch(batch *jaeger.Batch) (err error)
}

// Reporter 使用grpc 协议将span 数据上报到collector
type Reporter struct {
    // grpc 客户端
    collector api_v2.CollectorServiceClient
}

// ClientConfigManager 获取采样策略
type ClientConfigManager interface {
    GetSamplingStrategy(serviceName string) (*sampling.SamplingStrategyResponse, error)
}

// SamplingManager 从collector 获取采集抽样策略
type SamplingManager struct {
    client api_v2.SamplingManagerClient
}

// Builder 结构体用来保存Agent 配置
type Builder struct {
    // ProcessorConfiguration 处理器配置
    Processors []ProcessorConfiguration `yaml:"processors"`
    // HTTPServerConfiguration 包含采样策略服务的配置
    HTTPServer HTTPServerConfiguration  `yaml:"httpServer"`
    reporters []reporter.Reporter
}

// Agent 服务结构
type Agent struct {
    // 上报数据处理
    processors []processors.Processor
    // 采样策略处理
    httpServer *http.Server
}
```

创建服务：

```go
// github.com/jaegertracing/jaeger/cmd/agent/main.go
// grpc 连接配置
grpcBuilder := grpc.NewConnBuilder().InitFromViper(v)
// grpc 客户端，用于与 Collector 交互
builders := map[reporter.Type]app.CollectorProxyBuilder{
    reporter.GRPC: app.GRPCCollectorProxyBuilder(grpcBuilder),
}
// 实例化grpc 客户端
cp, err := app.CreateCollectorProxy(..., builders)

// 创建agent 服务，接收应用程序的spans 数据
builder := new(app.Builder).InitFromViper(v)
agent, err := builder.CreateAgent(cp, logger, mFactory)

// 运行agent
agent.Run()
```


## 2. 创建 Collector 客户端

创建与 Collector 交互的客户端，将接收到的数据发送到collector。

### 2.1 创建客户端实例

创建 grpc 连接。

```go
// NewConnBuilder 创建grpc 连接创建器
func NewConnBuilder() *ConnBuilder {
    return &ConnBuilder{}
}

// github.com/jaegertracing/jaeger/cmd/agent/app/reporter/grpc/builder.go
// CreateConnection 创建grpc 连接
func (b *ConnBuilder) CreateConnection(logger *zap.Logger) (*grpc.ClientConn, error) {
    // ...
    // 基于ip 端口创建grpc 连接
    dialTarget = b.CollectorHostPorts[0]
    
    // 开启grpc 连接
    return grpc.Dial(dialTarget, dialOptions...)
}
```

`CreateCollectorProxy` 创建到Collector 的代理层。

```go
// 包装grpc 协议的上报器，用于将数据上报到collector
func GRPCCollectorProxyBuilder(builder *grpc.ConnBuilder) CollectorProxyBuilder {
    return func(opts ProxyBuilderOptions) (proxy CollectorProxy, err error) {
        return grpc.NewCollectorProxy(builder, opts.AgentTags, opts.Metrics, opts.Logger)
    }
}


// github.com/jaegertracing/jaeger/cmd/agent/app/builder.go
// CreateCollectorProxy 创建上报代理
func CreateCollectorProxy(...) (CollectorProxy, error) {
    builder, ok := builders[opts.ReporterType]

    return builder(opts)
}
```

`NewCollectorProxy` 基于grpc 协议的连接客户端。

```go
// github.com/jaegertracing/jaeger/cmd/agent/app/reporter/grpc/collector_proxy.go
// NewCollectorProxy 创建一个grpc 创建器
func NewCollectorProxy(builder *ConnBuilder, agentTags map[string]string, mFactory metrics.Factory, logger *zap.Logger) (*ProxyBuilder, error) {
    // grpc 连接
    conn, err := builder.CreateConnection(logger)

    // 创建上报器实例
    r1 := NewReporter(conn, agentTags, logger)
    // r2, r3 是上报器中间件

    // ...
    return &ProxyBuilder{
        conn:     conn,
        reporter: r3,
        // 创建获取采样策略的客户端
        manager:  configmanager.WrapWithMetrics(grpcManager.NewConfigManager(conn), grpcMetrics),
    }, nil
}

// GetReporter 实现CollectorProxy 接口
func (b ProxyBuilder) GetReporter() reporter.Reporter {
    return b.reporter
}

// 获取采样配置的客户端
func (b ProxyBuilder) GetManager() configmanager.ClientConfigManager {
    return b.manager
}
```

### 2.2 创建上报客户端

数据上报，将span 数据上报到collector。

```go
// github.com/jaegertracing/jaeger/cmd/agent/app/reporter/reporter.go
// NewReporter 创建grpc 客户端
func NewReporter(conn *grpc.ClientConn, agentTags map[string]string, logger *zap.Logger) *Reporter {
    return &Reporter{
        collector: api_v2.NewCollectorServiceClient(conn),
        // ...
    }
}

// EmitBatch 实现上报器的 EmitBatch() 接口
func (r *Reporter) EmitBatch(b *thrift.Batch) error {
    return r.send(jConverter.ToDomain(b.Spans, nil), jConverter.ToDomainProcess(b.Process))
}

// 通过grpc 发送数据到collector
func (r *Reporter) send(spans []*model.Span, process *model.Process) error {
    batch := model.Batch{Spans: spans, Process: process}
    req := &api_v2.PostSpansRequest{Batch: batch}
    _, err := r.collector.PostSpans(context.Background(), req)
    // ...
    return err
}
```

### 2.3 采样配置客户端

采样配置管理客户端，用于获取采集策略。

```go
// github.com/jaegertracing/jaeger/cmd/agent/app/configmanager/manager.go
// NewConfigManager 创建基于grpc 协议的采集抽样管理器
func NewConfigManager(conn *grpc.ClientConn) *SamplingManager {
    return &SamplingManager{
        client: api_v2.NewSamplingManagerClient(conn),
    }
}

// GetSamplingStrategy 获取采集抽样数
func (s *SamplingManager) GetSamplingStrategy(serviceName string) (*sampling.SamplingStrategyResponse, error) {
    r, err := s.client.GetSamplingStrategy(context.Background(), &api_v2.SamplingStrategyParameters{ServiceName: serviceName})
    // ...
    return jaeger.ConvertSamplingResponseFromDomain(r)
}
```

## 3. 创建Agent 服务

创建Agent 服务，接收应用传送过来的spans 数据；提供采样策略获取接口。

```go
// 创建agent 实例
agent, err := builder.CreateAgent(cp, logger, mFactory)

// github.com/jaegertracing/jaeger/cmd/agent/app/builder.go
// CreateAgent 创建Agent 实例
func (b *Builder) CreateAgent(primaryProxy CollectorProxy, logger *zap.Logger, mFactory metrics.Factory) (*Agent, error) {
    // Collector 上报客户端
    r := b.getReporter(primaryProxy)
    
    // 上报数据处理逻辑
    processors, err := b.getProcessors(r, mFactory, logger)

    // 采样策略获取逻辑
    server := b.HTTPServer.getHTTPServer(primaryProxy.GetManager(), mFactory)
    return NewAgent(processors, server, logger), nil
}

// 获取上报器
// 这里获取的是到Collector 的上报客户端
func (b *Builder) getReporter(primaryProxy CollectorProxy) reporter.Reporter {
    // ...
    rep[0] = primaryProxy.GetReporter()
    return reporter.NewMultiReporter(rep...)
}
```

创建Agent 实例：

```go
// NewAgent 创建一个新的 Agent
func NewAgent(
    processors []processors.Processor,
    httpServer *http.Server,
    logger *zap.Logger,
) *Agent {
    a := &Agent{
        processors: processors,
        httpServer: httpServer,
    }
    a.httpAddr.Store("")
    return a
}
```


## 4. 采样策略服务

创建http 服务，提供获取采样策略的接口。

```go
// github.com/jaegertracing/jaeger/cmd/agent/app/builder.go
// GetHTTPServer 创建http 服务，用来查询采样策略
func (c HTTPServerConfiguration) getHTTPServer(manager configmanager.ClientConfigManager, mFactory metrics.Factory) *http.Server {
    // ...
    return httpserver.NewHTTPServer(c.HostPort, manager, mFactory)
}

// NewHTTPServer 创建http 服务，注册路由端点
func NewHTTPServer(hostPort string, manager configmanager.ClientConfigManager, mFactory metrics.Factory) *http.Server {
    handler := clientcfghttp.NewHTTPHandler(...)

    // 注册http 路由
    r := mux.NewRouter()
    handler.RegisterRoutes(r)
    return &http.Server{Addr: hostPort, Handler: r}
}

func (h *HTTPHandler) RegisterRoutes(router *mux.Router) {
    // ...
    router.HandleFunc(prefix+"/sampling", func(w http.ResponseWriter, r *http.Request) {
        h.serveSamplingHTTP(w, r, false)
    }).Methods(http.MethodGet)
}

// 获取采样策略信息
func (h *httpHandler) serveSamplingHTTP(w http.ResponseWriter, r *http.Request, thriftEnums092 bool) {
    service, err := h.serviceFromRequest(w, r)
    
    // 从collector 获取采样策略
    resp, err := h.params.ConfigManager.GetSamplingStrategy(service)
    // 数据序列化为json
    jsonBytes, err := json.Marshal(resp)
    if err = h.writeJSON(w, jsonBytes); err != nil {
        return
    }
}
```

## 5. 开启服务

开启Agent 服务。

```go
// github.com/jaegertracing/jaeger/cmd/agent/app/agent.go
// Run 运行所有的http 和udp 服务
func (a *Agent) Run() error {
    listener, err := net.Listen("tcp", a.httpServer.Addr)
    
    // 开启http 服务，用于获取agent 采样策略
    go func() {
        a.httpServer.Serve(listener)
    }()

    // 开启不同协议的处理器服务
    for _, processor := range a.processors {
        go processor.Serve()
    }
    return nil
}
```

## 参考资料

- github.com/jaegertracing/jaeger/cmd/agent/main.go

