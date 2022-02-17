<!-- ---
title: jaeger
date: 2019-07-09 15:02:35
category: src, jaeger
--- -->

# Jaeger Client Go 实现

> 客户端引入golang 语言Jaeger Client SDK 后的处理流程。

## 1. jaeger client 使用方式

Jaeger 在应用中一般以中间件的形式进行采集，中间件分别附加在服务端服务和客户端调用上。

服务端中间件负责记录当前服务调用的span 信息及其耗时。

客户端中间件在调用远程服务时，创建span，调用完成后上报 span 信息。

中间件中调用步骤：

1. 初始化Tracer 实例，创建全局tracer，一般只创建一次 `cfg.NewTracer()`
2. 针对每个调用请求创建span，基于Tracer 创建span `tracer.StartSpan()`
3. 添加span 信息，设置请求的span 信息 `span.SetTag()`
4. 请求处理完成，触发 `span.Finish()`
5. 使用上报器上报span 信息 `reporter.Report()`
6. 异步将span 信息提交到收集器

`span.Finish` 操作会将span 推给report，由report 将数据上报到收集器中，Sample 采样器会判断当前span 是否需要采集，如果需要采集，才会调用 reporter 上报。

![](images/jaeger_client_go.svg)

## 2. 创建tracer

主要数据结构：

```go
// Configuration 创建Tracer 的配置
type Configuration struct {
    ServiceName string `yaml:"serviceName"` // 服务名
    Tags []opentracing.Tag `yaml:"tags"` // tracer 默认标签
    Sampler             *SamplerConfig             `yaml:"sampler"` // 采样配置
    Reporter            *ReporterConfig            `yaml:"reporter"` // 上报配置
    Throttler           *ThrottlerConfig           `yaml:"throttler"` // 限流配置
}

// Tracer 结构
// Tracer 需要实现 opentracing.Tracer 接口
type Tracer struct {
    serviceName string
    sampler  Sampler // 采集器
    reporter Reporter // 上报器
    metrics  Metrics // 指标收集
    // ...
}

// opentracing.Tracer 接口
type Tracer interface {
    // Create, start, and return a new Span with the given `operationName` and
    // incorporate the given StartSpanOption `opts`
    StartSpan(operationName string, opts ...StartSpanOption) Span

    // Inject() takes the `sm` SpanContext instance and injects it for
    // propagation within `carrier`. 
    Inject(sm SpanContext, format interface{}, carrier interface{}) error

    // Extract() returns a SpanContext instance given `format` and `carrier`.
    Extract(format interface{}, carrier interface{}) (SpanContext, error)
}

// Span 结构体，需要实现 opentracing.Span 接口
type Span struct {
    tracer *Tracer
    context SpanContext
    operationName string
    startTime time.Time
    duration time.Duration
    tags []Tag
    references []Reference
    // ...
}

// opentracing.Span 接口
type Span interface {
    // 设置最后操作时间错并且结束Span
    Finish()
    // 返回创建span 的Tracer
    Tracer() Tracer
    // 添加标签键值对
    SetTag(key string, value interface{}) Span
}
```

创建`Tracer` 实例：

```go
// github.com/jaegertracing/jaeger-client-go/config/example_test.go
// 客户端引入trace 时，创建一个tracer，此处为tracer 入口
tracer, closer, err := cfg.NewTracer()

// github.com/jaegertracing/jaeger-client-go/config/config.go
// 基于配置和传入的选项创建一个Tracer 实例。
func (c Configuration) NewTracer(options ...Option) (opentracing.Tracer, io.Closer, error) {
    // ...

    // 创建采样器
    sampler := opts.sampler
    if sampler == nil {
        s, err := c.Sampler.NewSampler(c.ServiceName, tracerMetrics)
        sampler = s
    }

    // 创建上报器
    reporter := opts.reporter
    if reporter == nil {
        r, err := c.Reporter.NewReporter(c.ServiceName, tracerMetrics, opts.logger)
        reporter = r
    }

    // ...
    // 创建tracer 实例
    tracer, closer := jaeger.NewTracer(
        c.ServiceName,
        sampler,
        reporter,
        tracerOptions...,
    )

    return tracer, closer, nil
}
```

创建Tracer 实例：

```go
// NewTracer 
func NewTracer(
    serviceName string,
    sampler Sampler,
    reporter Reporter,
    options ...TracerOption,
) (opentracing.Tracer, io.Closer) {
    t := &Tracer{
        serviceName:   serviceName,
        sampler:       sampler,
        reporter:      reporter,
        injectors:     make(map[interface{}]Injector),
        extractors:    make(map[interface{}]Extractor),
        metrics:       *NewNullMetrics(),
        spanAllocator: simpleSpanAllocator{},
    }

    // ...
    return t, t
}
```

创建采样器：

```go
s, err := c.Sampler.NewSampler(c.ServiceName, tracerMetrics)

// NewSampler 创建基于配置的采样器
func (sc *SamplerConfig) NewSampler(
    serviceName string,
    metrics *jaeger.Metrics,
) (jaeger.Sampler, error) {
    samplerType := strings.ToLower(sc.Type)
    // 采样率采样
    if samplerType == jaeger.SamplerTypeProbabilistic {
        if sc.Param >= 0 && sc.Param <= 1.0 {
            return jaeger.NewProbabilisticSampler(sc.Param)
        }
    }
        
    // 限流采样
    if samplerType == jaeger.SamplerTypeRateLimiting {
        return jaeger.NewRateLimitingSampler(sc.Param), nil
    }
    
    // ...
    return nil, fmt.Errorf("unknown sampler type (%s)", sc.Type)
}

// github.com/jaegertracing/jaeger-client-go/sampler.go
// 采样率采样
func NewProbabilisticSampler(samplingRate float64) (*ProbabilisticSampler, error) {
    // ...
    return newProbabilisticSampler(samplingRate), nil
}

func newProbabilisticSampler(samplingRate float64) *ProbabilisticSampler {
    s := new(ProbabilisticSampler)
    s.delegate = s.IsSampled
    return s.init(samplingRate)
}

// IsSampled 判断是否需要采集
func (s *ProbabilisticSampler) IsSampled(id TraceID, operation string) (bool, []Tag) {
    return s.samplingBoundary >= id.Low&maxRandomNumber, s.tags
}
```

## 3. 创建span

创建完Tracer 实例后，调用Tracer 的`StartSpan` 创建span。

```go
// github.com/jaegertracing/jaeger-client-go/tracer.go
// 创建span 实例
func (t *Tracer) StartSpan(
    operationName string,
    options ...opentracing.StartSpanOption,
) opentracing.Span {
    sso := opentracing.StartSpanOptions{}
    for _, o := range options {
        o.Apply(&sso)
    }
    return t.startSpanWithOptions(operationName, sso)
}

func (t *Tracer) startSpanWithOptions(
    operationName string,
    options opentracing.StartSpanOptions,
) opentracing.Span {
    if options.StartTime.IsZero() {
        options.StartTime = t.timeNow()
    }

    // ...
    var references []Reference
    var parent SpanContext
    var ctx SpanContext

    // span 关系
    for _, ref := range options.References {
        ctxRef, ok := ref.ReferencedContext.(SpanContext)

        // ...

        references = append(references, Reference{Type: ref.Type, Context: ctxRef})
        if !hasParent {
            parent = ctxRef
            hasParent = ref.Type == opentracing.ChildOfRef
        }
    }

    // ...

    var internalTags []Tag
    newTrace := false
    if !isSelfRef {
        if !hasParent || !parent.IsValid() {
            newTrace = true
            ctx.traceID.Low = t.randomID()
            
            ctx.spanID = SpanID(ctx.traceID.Low)
            ctx.parentID = 0
            // ...
        } else {
            ctx.traceID = parent.traceID
            if rpcServer && t.options.zipkinSharedRPCSpan {
                // Support Zipkin's one-span-per-RPC model
                ctx.spanID = parent.spanID
                ctx.parentID = parent.parentID
            } 
            // ...
        }
        // ...
    }

    sp := t.newSpan()
    sp.context = ctx
    sp.tracer = t
    sp.operationName = operationName
    sp.startTime = options.StartTime
    sp.references = references

    // ...
    // 采样器判断是否需要采集
    if !sp.isSamplingFinalized() {
        // 判断是否需要采集
        decision := t.sampler.OnCreateSpan(sp)
        // 设置采样状态
        sp.applySamplingDecision(decision, false)
    }

    return sp
}

// newSpan 分配一个span 对象
func (t *Tracer) newSpan() *Span {
    // &Span{} 返回空的Span 结构体
    return t.spanAllocator.Get()
}
```

采样器调用实现:

```go
// 采样器判断是否需要采样
func (s *legacySamplerV1Base) OnCreateSpan(span *Span) SamplingDecision {
    isSampled, tags := s.delegate(span.context.traceID, span.operationName)
    return SamplingDecision{Sample: isSampled, Retryable: false, Tags: tags}
}

// 设置采样状态
func (s *Span) applySamplingDecision(decision SamplingDecision, lock bool) {
    // ...
    if decision.Sample {
        s.context.samplingState.setSampled()
    }
}
```

### 3.1 设置Span Tag 信息

调用span 的`SetTag` 函数设置span 信息

```go
// github.com/jaegertracing/jaeger-client-go/span.go
// SetTag implements SetTag() of opentracing.Span
func (s *Span) SetTag(key string, value interface{}) opentracing.Span {
    return s.setTagInternal(key, value, true)
}

func (s *Span) setTagInternal(key string, value interface{}, lock bool) opentracing.Span {
    // 判断是否完成，或者采集
    if s.isWriteable() {
        if lock {
            s.Lock()
            defer s.Unlock()
        }
        s.appendTagNoLocking(key, value)
    }
    return s
}

func (s *Span) appendTagNoLocking(key string, value interface{}) {
    s.tags = append(s.tags, Tag{key: key, value: value})
}
```

### 3.2 业务请求处理完成上报span

业务请求处理完成后触发`span.Finish` 上报span

```go
// github.com/jaegertracing/jaeger-client-go/span.go
// Finish span 完成
func (s *Span) Finish() {
    s.FinishWithOptions(opentracing.FinishOptions{})
}

// FinishWithOptions implements opentracing.Span API
func (s *Span) FinishWithOptions(options opentracing.FinishOptions) {
    if options.FinishTime.IsZero() {
        options.FinishTime = s.tracer.timeNow()
    }

    // 通知观察者
    s.observer.OnFinish(options)

    // ...
    s.tracer.reportSpan(s)
}
```

调用tracer 的`reportSpan` 上报span。

上报器将span 发送到收集器。

```go
// github.com/jaegertracing/jaeger-client-go/tracer.go
func (t *Tracer) reportSpan(sp *Span) {
    // 确定是否需要采样
    if sp.context.IsSampled() {
        t.reporter.Report(sp)
    }

    // 重置当前span 实例，方便下次请求复用
    sp.Release()
}
```

## 参考资料

- github.com/jaegertracing/jaeger-client-go/config/config.go
