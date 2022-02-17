<!-- ---
title: jaeger query
date: 2019-07-11 10:34:46
category: src, jaeger
--- -->

# Jaeger Query 实现

> jaeger query 从底层存储中查询traces 和span 数据。

jaeger query 结构图：

1. 创建存储实例和span 读取器
2. 创建查询处理服务
    1. 查询服务调用spanReader 读取trace 数据
3. 创建网络监听服务
    1. 创建Server 实例
    2. 创建http 服务
4. 开启服务

![](images/jaeger_query.svg)

## 1. query 服务入口

主要数据结构：

```go
// Reader 读取器接口
type Reader interface {
    GetTrace(ctx context.Context, traceID model.TraceID) (*model.Trace, error)
    GetServices(ctx context.Context) ([]string, error)
    GetOperations(ctx context.Context, query OperationQueryParameters) ([]Operation, error)
    FindTraces(ctx context.Context, query *TraceQueryParameters) ([]*model.Trace, error)
    FindTraceIDs(ctx context.Context, query *TraceQueryParameters) ([]model.TraceID, error)
}

// APIHandler 实现http 协议的查询服务
type APIHandler struct {
    queryService *querysvc.QueryService
    queryParser  queryParser
    // ...
}
```

1. 创建存储实例和span 读取器
2. 创建查询处理服务
3. 创建网络监听服务
4. 开启服务

```go
// github.com/jaegertracing/jaeger/cmd/query/main.go
// 创建存储实例
storageFactory, err := storage.NewFactory(storage.FactoryConfigFromEnvAndCLI(os.Args, os.Stderr))
// span 读取器
spanReader, err := storageFactory.CreateSpanReader()

// 创建查询处理服务
queryService := querysvc.NewQueryService(
    spanReader,
    dependencyReader,
    *queryServiceOptions)

// 创建网络监听服务
server := app.NewServer(svc, queryService, queryOpts, tracer)

// 开启服务
server.Start()
```

创建存储查询器：

```go
// CreateSpanReader 取出配置的storage 实例，并且创建对应的SpanReader 实例
func (f *Factory) CreateSpanReader() (spanstore.Reader, error) {
    factory, ok := f.factories[f.SpanReaderType]
    // ...
    return factory.CreateSpanReader()
}
```

## 2. 创建查询处理实例

查询请求最终请求读取器获取数据。

```go
// 创建查询服务
func NewQueryService(spanReader spanstore.Reader, dependencyReader dependencystore.Reader, options QueryServiceOptions) *QueryService {
    qsvc := &QueryService{
        spanReader:       spanReader,
        dependencyReader: dependencyReader,
        options:          options,
    }

    // ...
    return qsvc
}

// 处理查询trace 逻辑
func (qs QueryService) GetTrace(ctx context.Context, traceID model.TraceID) (*model.Trace, error) {
    // 通过spanReader 查询trace
    trace, err := qs.spanReader.GetTrace(ctx, traceID)
    return trace, err
}

// GetServices
func (qs QueryService) GetServices(ctx context.Context) ([]string, error) {
    return qs.spanReader.GetServices(ctx)
}

// GetOperations
func (qs QueryService) GetOperations(
    ctx context.Context,
    query spanstore.OperationQueryParameters,
) ([]spanstore.Operation, error) {
    return qs.spanReader.GetOperations(ctx, query)
}
```


## 3. 创建查询服务

开启http 和grpc 服务，接收查询请求，调用`queryService` 查询数据，然后返回查询结果数据。

1. 创建Server 实例
2. 创建http 服务
3. 开启服务

### 3.1 创建服务实例

```go
// github.com/jaegertracing/jaeger/cmd/query/app/server.go
// 创建并且初始化实例
func NewServer(svc *flags.Service, querySvc *querysvc.QueryService, options *QueryOptions, tracer opentracing.Tracer) *Server {
    return &Server{
        svc:          svc,
        querySvc:     querySvc,
        grpcServer:   createGRPCServer(querySvc, svc.Logger, tracer),
        httpServer:   createHTTPServer(querySvc, options, tracer, svc.Logger),
    }
}
```

### 3.2 创建HTTP 协议服务

```go
// 创建http 服务
createHTTPServer(querySvc *querysvc.QueryService, queryOpts *QueryOptions, tracer opentracing.Tracer, logger *zap.Logger) *http.Server {
    // http 请求处理器
    apiHandler := NewAPIHandler(
        querySvc,
        apiHandlerOptions...)
    
    // 注册路由，路由注册到处理器上
    r := NewRouter()
    apiHandler.RegisterRoutes(r)
    
    // 注册静态路由
    RegisterStaticHandler(r, logger, queryOpts)

    var handler http.Handler = r
    // 返回http 服务实例
    return &http.Server{
        Handler: recoveryHandler(handler),
    }
}
```

### 3.3 注册http 请求处理handler

```go
// NewAPIHandler 创建http 请求处理器
func NewAPIHandler(queryService *querysvc.QueryService, options ...HandlerOption) *APIHandler {
    aH := &APIHandler{
        queryService: queryService,
        // ...
    }
    // ...
    return aH
}

// RegisterRoutes 注册路由
func (aH *APIHandler) RegisterRoutes(router *mux.Router) {
    aH.handleFunc(router, aH.getTrace, "/traces/{%s}", traceIDParam).Methods(http.MethodGet)
    aH.handleFunc(router, aH.search, "/traces").Methods(http.MethodGet)
    aH.handleFunc(router, aH.getServices, "/services").Methods(http.MethodGet)
    // ...
}

// getTrace 处理traces 查询请求
// 这里解析请求的trace id
// 通过QueryService 查询这个trace
// 将数据格式化为方便前端显示，最后响应给前端
func (aH *APIHandler) getTrace(w http.ResponseWriter, r *http.Request) {
    traceID, ok := aH.parseTraceID(w, r)

    // 通过QueryService 查询这个trace
    trace, err := aH.queryService.GetTrace(r.Context(), traceID)
    
    uiTrace, uiErr := aH.convertModelToUI(trace, shouldAdjust(r))
    structuredRes := structuredResponse{
        Data: []*ui.Trace{
            uiTrace,
        },
        Errors: uiErrors,
    }
    
    // 结果响应给前端
    aH.writeJSON(w, r, &structuredRes)
}

```

## 4. 启动服务

启动grpc 和http 服务。

```go
// 开启http 和grpc 服务
// 这里用到了 cmux ，可以将同一个端口上的不同请求转发给相应协议处理
func (s *Server) Start() error {
    // 开启tpc 监听
    conn, err := net.Listen("tcp", fmt.Sprintf(":%d", s.queryOptions.Port))
    s.conn = conn

    // cmux 作为反向代理，根据请求内容将请求转发给http 或者grpc 服务
    cmuxServer := cmux.New(s.conn)

    // grpc 监听代理
    grpcListener := cmuxServer.MatchWithWriters(
        cmux.HTTP2MatchHeaderFieldSendSettings("content-type", "application/grpc"),
        cmux.HTTP2MatchHeaderFieldSendSettings("content-type", "application/grpc+proto"),
    )

    // http 监听代理
    httpListener := cmuxServer.Match(cmux.Any())

    go func() {
        // 开启http 服务
        s.httpServer.Serve(httpListener)
        // ...
    }()

    // 开启cmux 代理服务
    go func() {
        err := cmuxServer.Serve()
        // ...
    }()

    return nil
}

```

## 参考资料

- github.com/jaegertracing/jaeger/cmd/query/main.go

