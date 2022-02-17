<!-- ---
title: jaeger collector storage
date: 2020-04-23 15:41:53
category: showcode, jaeger
--- -->

# Jaeger Collector 存储实现

> 落地存储trace 和span 数据实现逻辑，来与存储和查询链路数据。


主要结构：

1. 创建存储实例
2. 存储逻辑实例实现
3. 底层存储器实现

![](images/jaeger_collector_storage.svg)


## 1. 创建存储器实例

`storageFactory` 聚合存储层，提供接口将收集到的span 数据写入底层存储服务中，或者从底层存储中查询trace 数据。聚合存储层会根据配置的底层存储器类型，会初始化对应的存储实例，再调用底层存储实例完成存储和查询功能。

存储层和底层存储都会需要实现`Factory` 接口，存储层可以聚合多个底层存储，在存储数据时会依次调用进行存储。

1. 创建对应的底层存储实例
2. span 写入逻辑
3. span 读取逻辑

主要数据结构：

```go
// github.com/jaegertracing/jaeger/plugin/storage/factory_config.go
// 存储处理器需要实现的接口
type Factory interface {
    // Initialize 执行存储需要进行的初始化操作
    Initialize(metricsFactory metrics.Factory, logger *zap.Logger) error
    // 返回读取span 数据的实例，需要实现spanstore.Reader 接口
    CreateSpanReader() (spanstore.Reader, error)
    // 返回写入span 数据的实例，需要实现spanstore.Writer 接口
    CreateSpanWriter() (spanstore.Writer, error)
    // 依赖读取
    CreateDependencyReader() (dependencystore.Reader, error)
}

// 从环境变量或者终端参数获取存储配置
// Factory 存储服务实例工厂
type Factory struct {
    FactoryConfig
    metricsFactory metrics.Factory
    factories      map[string]storage.Factory // 收集配置的多个底层存储实例
}

// Reader 读取器需要实现的接口，用于查找和读取trace
type Reader interface {
    GetTrace(ctx context.Context, traceID model.TraceID) (*model.Trace, error)
    GetServices(ctx context.Context) ([]string, error)
    GetOperations(ctx context.Context, service string) ([]string, error)
    FindTraces(ctx context.Context, query *TraceQueryParameters) ([]*model.Trace, error)
    FindTraceIDs(ctx context.Context, query *TraceQueryParameters) ([]model.TraceID, error)
}

// Factory 内存存储实例结构，span 数据存储在Store 结构中
type Factory struct {
    options        Options
    metricsFactory metrics.Factory
    logger         *zap.Logger
    store          *Store
}

type Store struct {
    ids        []*model.TraceID
    traces     map[model.TraceID]*model.Trace
    // ...
}

// Writer spanWriter 实例需要实现的接口。
type Writer interface {
    WriteSpan(span *model.Span) error
}
```

### 1.1 创建底层存储实例

存储接口层，初始化时会创建配置的底层存储实例。

```go
// github.com/jaegertracing/jaeger/cmd/collector/main.go
// 创建存储实例
storageFactory, err := storage.NewFactory(storage.FactoryConfigFromEnvAndCLI(os.Args, os.Stderr))
storageFactory.InitFromViper(v)
// 存储初始化，用于初始化底层连接
storageFactory.Initialize(baseFactory, logger)
spanWriter, err := storageFactory.CreateSpanWriter()
```

```go
// github.com/jaegertracing/jaeger/plugin/storage/factory.go
// 解析数据存储方式
storageFactory, err := storage.NewFactory(storage.FactoryConfigFromEnvAndCLI(os.Args, os.Stderr))

// NewFactory 创建低层存储服务实例，实现了Factory 接口，代理处理请求到底层存储实例
func NewFactory(config FactoryConfig) (*Factory, error) {
    // ...
    for _, storageType := range f.SpanWriterTypes {
        uniqueTypes[storageType] = struct{}{}
    }

    // 收集存储类型，一般只有一种
    f.factories = make(map[string]storage.Factory)
    for t := range uniqueTypes {
        ff, err := f.getFactoryOfType(t)
        // ...
        f.factories[t] = ff
    }
    return f, nil
}

// 支持的存储类型
func (f *Factory) getFactoryOfType(factoryType string) (storage.Factory, error) {
    switch factoryType {
    case elasticsearchStorageType:
        return es.NewFactory(), nil
    case memoryStorageType:
        return memory.NewFactory(), nil
    }
}
```

Initialize 初始化，存储层的初始化函数会依次调用底层存储实例的初始化。

```go
// 存储初始化，用于初始化底层连接
storageFactory.Initialize(baseFactory, logger)

// 存储层的初始化函数会依次调用底层存储实例的初始化
func (f *Factory) Initialize(metricsFactory metrics.Factory, logger *zap.Logger) error {
    // ...
    for _, factory := range f.factories {
        factory.Initialize(metricsFactory, logger)
    }
    return nil
}
```


### 1.2 创建写入器

创建span 写入实例。存储层`CreateSpanWriter` 函数实际是调用底层存储实例`CreateSpanWriter` 函数来创建`spanWriter` 写入器。

```go
// github.com/jaegertracing/jaeger/plugin/storage/factory.go
// CreateSpanWriter
func (f *Factory) CreateSpanWriter() (spanstore.Writer, error) {
    var writers []spanstore.Writer
    // 获取所有底层存储的SpanWriter 实例
    for _, storageType := range f.SpanWriterTypes {
        factory, ok := f.factories[storageType]
        // ...
        // factory 代码一种底层存储实体
        writer, err := factory.CreateSpanWriter()

        writers = append(writers, writer)
    }

    spanWriter = writers[0]
    return spanWriter, nil
}
```


### 1.3 创建读取器

```go
//github.com/jaegertracing/jaeger/storage/spanstore/interface.go
// CreateSpanReader 读取时只需要从一个底层存储读取即可
func (f *Factory) CreateSpanReader() (spanstore.Reader, error) {
    factory, ok := f.factories[f.SpanReaderType]
    // ...
    return factory.CreateSpanReader()
}
```

## 2. Memory 存储实现

### 2.1 创建Memory 存储实例

创建内存存储实例

```go
memory.NewFactory()

//github.com/jaegertracing/jaeger/plugin/storage/memory/factory.go
// NewFactory 创建一个内存实例
func NewFactory() *Factory {
    return &Factory{}
}
```

### 2.2 创建内存存储写入器实现

对于内存类型存储，CreateSpanWriter 实现如下：

```go
// Initialize 初始化存储处理
func (f *Factory) Initialize(metricsFactory metrics.Factory, logger *zap.Logger) error {
    f.store = WithConfiguration(f.options.Configuration)
    return nil
}

// github.com/jaegertracing/jaeger/plugin/storage/memory/factory.go
// CreateSpanWriter implements storage.Factory
func (f *Factory) CreateSpanWriter() (spanstore.Writer, error) {
    return f.store, nil
}

// f.store 需要实现 Writer 接口的WriteSpan 函数
// github.com/jaegertracing/jaeger/plugin/storage/memory/memory.go
// WriteSpan writes the given span
func (m *Store) WriteSpan(span *model.Span) error {
    // ...
    if _, ok := m.traces[span.TraceID]; !ok {
        m.traces[span.TraceID] = &model.Trace{}
        // ...
    }

    m.traces[span.TraceID].Spans = append(m.traces[span.TraceID].Spans, span)

    return nil
}
```


### 2.3 创建内存存储读取器

创建读取器。

```go
//github.com/jaegertracing/jaeger/plugin/storage/memory/factory.go
// CreateSpanReader implements storage.Factory
func (f *Factory) CreateSpanReader() (spanstore.Reader, error) {
    return f.store, nil
}

// GetTrace 获取trace 数据
func (m *Store) GetTrace(ctx context.Context, traceID model.TraceID) (*model.Trace, error) {
    retMe := m.traces[traceID]
    
    return m.copyTrace(trace), nil
}

// Spans 数据复制一份返回前端
func (m *Store) copyTrace(trace *model.Trace) *model.Trace {
    return &model.Trace{
        Spans:    append([]*model.Span(nil), trace.Spans...),
        Warnings: append([]string(nil), trace.Warnings...),
    }
}
```


## 参考资料

- github.com/jaegertracing/jaeger/storage/factory.go
- github.com/jaegertracing/jaeger/plugin/storage/memory/factory.go
