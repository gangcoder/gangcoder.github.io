<!-- ---
title: jaeger collector
date: 2019-07-11 10:34:30
category: src, jaeger
--- -->

# Jaeger Collector 实现

> jaeger collector 接收agent 和client 发送的span 数据，并且将这些数据写入存储中。

关键步骤：
1. 创建span 数据存储处理器
2. 创建采样策略存储处理器
3. 创建监听服务，接收spans 数据；接收采样策略获取请求
4. 将spans 数据写入底层存储


代码结构图：

1. 创建存储实例
2. 创建span 数据处理handler
3. 创建各协议服务监听
4. 开启服务

![](images/jaeger_collector.svg)


## 1. 开启collector 服务

主要数据结构：

```go
type Collector struct {
    // 实例名
    serviceName    string
    // 存储实例
    spanWriter     spanstore.Writer
    // 采样策略实例
    strategyStore  strategystore.StrategyStore
    spanProcessor  processor.SpanProcessor
    spanHandlers   *SpanHandlers
    // 服务
    hServer    *http.Server
    zkServer   *http.Server
    grpcServer *grpc.Server
}

// SpanProcessor 处理接口，用来处理spans 数据
type SpanProcessor interface {
    ProcessSpans(mSpans []*model.Span, options ProcessSpansOptions) ([]bool, error)
}

// span 处理程序
type spanProcessor struct {
    queue           *queue.BoundedQueue
    processSpan     ProcessSpan
    spanWriter      spanstore.Writer
    // ...
}

type jaegerBatchesHandler struct {
    modelProcessor processor.SpanProcessor
}
type JaegerBatchesHandler interface {
    // SubmitBatches 数据处理接口
    SubmitBatches(batches []*jaeger.Batch, options SubmitBatchOptions) ([]*jaeger.BatchSubmitResponse, error)
}

// APIHandler 处理Collector 的http 请求
type APIHandler struct {
    jaegerBatchesHandler JaegerBatchesHandler
}

type HTTPHandler struct {
    params  HTTPHandlerParams
}

type HTTPHandlerParams struct {
    ConfigManager  configmanager.ClientConfigManager // required
}
```

1. 创建存储器实例
2. 创建采样策略存储实例
3. 创建Collector 服务实例
4. 开启服务

```go
// github.com/jaegertracing/jaeger/cmd/collector/main.go
// span 数据存储方式
storageFactory, err := storage.NewFactory(storage.FactoryConfigFromEnvAndCLI(os.Args, os.Stderr))
// 创建存储实例
storageFactory.InitFromViper(v)
storageFactory.Initialize(baseFactory, logger)
spanWriter, err := storageFactory.CreateSpanWriter()

// 采样策略存储方式
strategyStoreFactory, err := ss.NewFactory(ss.FactoryConfigFromEnv())
// 创建采样策略实例
strategyStoreFactory.InitFromViper(v)
strategyStoreFactory.Initialize(metricsFactory, logger)
strategyStore, err := strategyStoreFactory.CreateStrategyStore()

// 创建Collector 实例
c := app.New(&app.CollectorParams{
    ServiceName:    serviceName,
    SpanWriter:     spanWriter,
    StrategyStore:  strategyStore,
})
// 启动Collector 服务
c.Start(collectorOpts)
```

创建服务实例：

```go
// 创建Collector 实例
func New(params *CollectorParams) *Collector {
    return &Collector{
        serviceName:    params.ServiceName,
        spanWriter:     params.SpanWriter,
        strategyStore:  params.StrategyStore,
    }
}
```

开启服务：

```go
func (c *Collector) Start(builderOpts *CollectorOptions) error {
    // 创建span 处理handler
    handlerBuilder, err := builder.NewSpanHandlerBuilder(
        spanWriter,
    )

    // 创建正对各协议的处理器
    c.spanProcessor = handlerBuilder.BuildSpanProcessor()
    c.spanHandlers = handlerBuilder.BuildHandlers(c.spanProcessor)

    // 启动grpc 协议服务
    grpcServer, err := server.StartGRPCServer(&server.GRPCServerParams{
        Handler:       c.spanHandlers.GRPCHandler,
        SamplingStore: c.strategyStore,
    });
    c.grpcServer = grpcServer

    // http 协议的处理服务
    httpServer, err := server.StartHTTPServer(&server.HTTPServerParams{
        Handler:        c.spanHandlers.JaegerBatchesHandler,
        SamplingStore:  c.strategyStore,
    })
    c.hServer = httpServer
}
```


## 2. 创建处理handler

创建处理器，用于将handler 解析出来的span 数据，存储底层存储。

```go
// 创建处理器，用于处理接收到的span 数据
c.spanProcessor = handlerBuilder.BuildSpanProcessor()

// github.com/jaegertracing/jaeger/cmd/collector/app/span_handler_builder.go
func (b *SpanHandlerBuilder) BuildSpanProcessor() processor.SpanProcessor {
    return NewSpanProcessor(
        b.SpanWriter,
        // ...
    )
}
```

创建不同协议的处理handler：

```go
// 创建不同协议的处理handler
// github.com/jaegertracing/jaeger/cmd/collector/main.go
c.spanHandlers = handlerBuilder.BuildHandlers(c.spanProcessor)

// github.com/jaegertracing/jaeger/cmd/collector/app/span_handler_builder.go
func (b *SpanHandlerBuilder) BuildHandlers(spanProcessor processor.SpanProcessor) *SpanHandlers {
    return &SpanHandlers{
        handler.NewZipkinSpanHandler(b.Logger, spanProcessor, ...),
        handler.NewJaegerSpanHandler(b.Logger, spanProcessor),
        handler.NewGRPCHandler(b.Logger, spanProcessor),
    }
}
```


### 2.1 创建存储写入处理器

创建 `SpanProcessor`，用于接收不同协议handler 解析的span 数据，并将数据写入存储实例。

开启队列消费操作。

具体分为：

1. 创建spanProcessor 处理器实例
2. 创建异步队列
3. 开启异步队列消费逻辑
4. 提供span 数据写入接口

```go
// github.com/jaegertracing/jaeger/cmd/collector/app/span_processor.go
// NewSpanProcessor 返回span 处理器
func NewSpanProcessor(
    spanWriter spanstore.Writer,
    opts ...Option,
) processor.SpanProcessor {
    // 创建span 处理器实例
    sp := newSpanProcessor(spanWriter, opts...)
    
    // 基于队列异步处理数据
    sp.queue.StartConsumers(sp.numWorkers, func(item interface{}) {
        value := item.(*queueItem)
        sp.processItemFromQueue(value)
    })
    return sp
}
```

创建 Span 处理器：

```go
func newSpanProcessor(spanWriter spanstore.Writer, opts ...Option) *spanProcessor {
    // 队列
    boundedQueue := queue.NewBoundedQueue(options.queueSize, droppedItemHandler)

    sp := spanProcessor{
        queue:           boundedQueue,
        spanWriter:      spanWriter,
        // ...
    }

    // 链式执行，分别执行预保存处理和保存处理
    processSpanFuncs := []ProcessSpan{options.preSave, sp.saveSpan}
    sp.processSpan = ChainedProcessSpan(processSpanFuncs...)
    return &sp
}

// github.com/jaegertracing/jaeger/pkg/queue/bounded_queue.go
// consumer 队列处理包装
func NewBoundedQueue(capacity int, onDroppedItem func(item interface{})) *BoundedQueue {
    return &BoundedQueue{
        capacity:      capacity,
        onDroppedItem: onDroppedItem,
        items:         make(chan interface{}, capacity),
        stopCh:        make(chan struct{}),
    }
}

// 保存处理实现，调用spanWriter 保存span 数据
func (sp *spanProcessor) saveSpan(span *model.Span) {
    // ...
    sp.spanWriter.WriteSpan(span)
}
```

span 消费逻辑实现：

```go
// StartConsumers 队列消费封装，最终调用consumer 函数
func (q *BoundedQueue) StartConsumers(num int, consumer func(item interface{})) {
    // ...
    q.consumer = consumer
    go func() {
        queue := *q.items
        for {
            select {
            case item, ok := <-queue:
                q.consumer(item)
            }
        }
    }()
    // ...
}

// consumer 消费数据
func (sp *spanProcessor) processItemFromQueue(item *queueItem) {
    // 先清理数据
    // 再保存数据
    sp.processSpan(sp.sanitizer(item.span))
}
```


### 2.2 Span 数据接收接口

格式化span 数据后，调用`SpanProcessor` 的`ProcessSpans` 接口将span 数据写入channel 。

```go
// github.com/jaegertracing/jaeger/cmd/collector/app/span_processor.go
func (sp *spanProcessor) ProcessSpans(mSpans []*model.Span, options processor.SpansOptions) ([]bool, error) {
    // ...
    for i, mSpan := range mSpans {
        ok := sp.enqueueSpan(mSpan, options.SpanFormat, options.InboundTransport)
    }
}

// github.com/jaegertracing/jaeger/cmd/collector/app/span_processor.go
// 最终调用队列的 Produce ，将数据添加到异步处理的channel 中
func (sp *spanProcessor) enqueueSpan(span *model.Span, originalFormat processor.SpanFormat, transport processor.InboundTransport) bool {
    item := &queueItem{
        queuedTime: time.Now(),
        span:       span,
    }

    // 将数据添加到queue
    return sp.queue.Produce(item)
}

// Produce 将数据写入队列的items 中
func (q *BoundedQueue) Produce(item interface{}) bool {
    // ...
    select {
    case *q.items <- item:
        return true
    }
}
```


### 2.3 创建 http 处理handler

创建 JaegerBatchesHandler 批量处理handler，将通过http 上报的span 数据写入队列中。

服务处理handler 需要实现`SubmitBatches` 接口用于保存span 数据。

```go
handler.NewJaegerSpanHandler(b.Logger, spanProcessor)

// NewJaegerSpanHandler 创建Jaeger 批量数据处理器
// github.com/jaegertracing/jaeger/cmd/collector/app/handler/thrift_span_handler.go
func NewJaegerSpanHandler(logger *zap.Logger, modelProcessor SpanProcessor) JaegerBatchesHandler {
    return &jaegerBatchesHandler{
        modelProcessor: modelProcessor,
    }
}

// http 接口接收的批量span 数据
func (jbh *jaegerBatchesHandler) SubmitBatches(batches []*jaeger.Batch, options SubmitBatchOptions) ([]*jaeger.BatchSubmitResponse, error) {
    // 响应数据
    for _, batch := range batches {
        mSpans := make([]*model.Span, 0, len(batch.Spans))
        for _, span := range batch.Spans {
            // span 数据格式转换
            mSpan := jConv.ToDomainSpan(span, batch.Process)
            mSpans = append(mSpans, mSpan)
        }

        // modelProcessor 处理span 数据，保存数据
        oks, err := jbh.modelProcessor.ProcessSpans(mSpans, processor.SpansOptions{
            InboundTransport: options.InboundTransport,
            SpanFormat:       processor.JaegerSpanFormat,
        })
    }
}
```

### 2.4 创建 grpc 处理handler

grpc 协议的处理器：

```go
handler.NewGRPCHandler(b.Logger, spanProcessor)

// 创建grpc 服务实例
func NewGRPCHandler(logger *zap.Logger, spanProcessor processor.SpanProcessor) *GRPCHandler {
    return &GRPCHandler{
        spanProcessor: spanProcessor,
    }
}

// PostSpans 实现grpc 服务端接口
func (g *GRPCHandler) PostSpans(ctx context.Context, r *api_v2.PostSpansRequest) (*api_v2.PostSpansResponse, error) {
    // 调用 spanProcessor 处理数据
    _, err := g.spanProcessor.ProcessSpans(r.GetBatch().Spans, processor.SpansOptions{
        InboundTransport: processor.GRPCTransport,
        SpanFormat:       processor.ProtoSpanFormat,
    })
}
```

## 3. 启动Grpc 协议服务

创建基于grpc 协议的服务。

开启Collector 的grpc 服务：

```go
server.StartGRPCServer(...)

// github.com/jaegertracing/jaeger/cmd/collector/app/server/grpc.go
func StartGRPCServer(params *GRPCServerParams) (*grpc.Server, error) {
    server = grpc.NewServer()

    // 监听网络
    listener, err := net.Listen("tcp", params.HostPort)
    // 开启服务
    serveGRPC(server, listener, params)
    return server, nil
}

func serveGRPC(server *grpc.Server, listener net.Listener, params *GRPCServerParams) error {
    // 注册服务端接口
    // 处理grpc 链路数据上报
    api_v2.RegisterCollectorServiceServer(server, params.Handler)
    // 处理grpc 采样策略获取
    api_v2.RegisterSamplingManagerServer(server, sampling.NewGRPCHandler(params.SamplingStore))

    // 开启服务
    go func() {
        server.Serve(listener)
    }()
}

// NewGRPCHandler 采样策略实例
func NewGRPCHandler(store strategystore.StrategyStore) GRPCHandler {
    return GRPCHandler{
        store: store,
    }
}

// GetSamplingStrategy 服务端采样策略获取接口实现
func (s GRPCHandler) GetSamplingStrategy(c context.Context, param *api_v2.SamplingStrategyParameters) (*api_v2.SamplingStrategyResponse, error) {
    // 采样策略服务接口实现
    r, err := s.store.GetSamplingStrategy(param.GetServiceName())
    // 响应数据
    return jaeger.ConvertSamplingResponseToDomain(r)
}
```

## 4. 启动HTTP 协议服务

创建基于http 协议的服务，通过http 协议获取远程传递过来的span 数据。

1. 注册http 端点路由
2. 开启http 服务

```go
// 开启http 协议服务
httpServer, err := server.StartHTTPServer(...)

func StartHTTPServer(params *HTTPServerParams) (*http.Server, error) {
    // 网络监听
    listener, err := net.Listen("tcp", params.HostPort)
    server := &http.Server{Addr: params.HostPort}
    
    // 开启服务
    serveHTTP(server, listener, params)
}

func serveHTTP(server *http.Server, listener net.Listener, params *HTTPServerParams) {
    // 注册http API 路由的处理handler
    r := mux.NewRouter()
    apiHandler := handler.NewAPIHandler(params.Handler)
    apiHandler.RegisterRoutes(r)

    // 注册采样策略处理handler
    cfgHandler := clientcfgHandler.NewHTTPHandler(clientcfgHandler.HTTPHandlerParams{
        ConfigManager: &clientcfgHandler.ConfigManager{
            // 获取采样策略
            SamplingStrategyStore: params.SamplingStore,
        },
        })
    cfgHandler.RegisterRoutes(r)

    // http 处理handler
    server.Handler = recoveryHandler(r)
    go func() {
        // 启动http 服务
        server.Serve(listener)
    }()
}
```

### 4.1 链路上报处理handler

创建http api handler。

注册路由端点和处理函数。当请求 `/api/traces` 端点时，调用`JaegerBatchesHandler` 将span 数据写入底层存储。

```go
// github.com/jaegertracing/jaeger/cmd/collector/app/http_handler.go
// NewAPIHandler 创建http 处理handler 实例
func NewAPIHandler(
    jaegerBatchesHandler JaegerBatchesHandler,
) *APIHandler {
    return &APIHandler{
        jaegerBatchesHandler: jaegerBatchesHandler,
    }
}

// RegisterRoutes 注册http 服务路由
func (aH *APIHandler) RegisterRoutes(router *mux.Router) {
    router.HandleFunc("/api/traces", aH.SaveSpan).Methods(http.MethodPost)
}

// 保存上报的链路数据
func (aH *APIHandler) saveSpan(w http.ResponseWriter, r *http.Request) {
    // 获取请求数据
    bodyBytes, err := ioutil.ReadAll(r.Body)

    // 解析请求数据
    batch := &tJaeger.Batch{}
    tdes.Read(batch, bodyBytes)
    batches := []*tJaeger.Batch{batch}
    
    // 调用存储实例接口，保存数据
    aH.jaegerBatchesHandler.SubmitBatches(batches, opts)
}
```

### 4.2 采样策略处理handler

创建采样策略接口，用于获取服务采样策略。

```go
// 创建采样策略处理handler
func NewHTTPHandler(params HTTPHandlerParams) *HTTPHandler {
    handler := &HTTPHandler{params: params}
    return handler
}

// 注册采样策略路由
func (h *HTTPHandler) RegisterRoutes(router *mux.Router) {
    router.HandleFunc(prefix+"/sampling", func(w http.ResponseWriter, r *http.Request) {
        h.serveSamplingHTTP(w, r, false /* thriftEnums092 */)
    }).Methods(http.MethodGet)
}

// 采样策略数据返回
func (h *HTTPHandler) serveSamplingHTTP(w http.ResponseWriter, r *http.Request, thriftEnums092 bool) {
    // 获取请求参数
    service, err := h.serviceFromRequest(w, r)
    
    // 获取服务对应的采样策略
    resp, err := h.params.ConfigManager.GetSamplingStrategy(service)
    
    // 返回采样策略数据
    jsonBytes, err := json.Marshal(resp)
    h.writeJSON(w, jsonBytes)
}
```

## 参考资料

- github.com/jaegertracing/jaeger/cmd/collector/main.go

