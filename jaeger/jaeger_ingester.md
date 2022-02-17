<!-- ---
title: jaeger ingester
date: 2019-07-12 21:16:26
category: src, jaeger
--- -->

# Jaeger Ingester 实现

> jaeger ingester 消费kafka 队列中的traces 数据，写入底层存储中。

关键逻辑：
1. 创建数据写入器
2. 创建ingester 消费者实例
   1. processor 消息处理器
   2. 创建kafka 客户端实例
   3. 消费逻辑实现
3. 开启consumer 实例
   1. 消息处理handler 实现
   2. 消息中间件
   3. 重试
   4. 提交
   5. 记录处理延迟
   6. 并发处理

![](images/jaeger_ingester.svg)


入口逻辑：

```go
// github.com/jaegertracing/jaeger/cmd/ingester/main.go
// 数据写入器
storageFactory, err := storage.NewFactory(storage.FactoryConfigFromEnvAndCLI(os.Args, os.Stderr))
spanWriter, err := storageFactory.CreateSpanWriter()

// 创建消费者实例
consumer, err := builder.CreateConsumer(logger, metricsFactory, spanWriter, options)

// 开启队列消费
consumer.Start()
```

## 1. 创建消费者实例

主要数据结构：

```go
// Span 处理器需要支持的接口
type SpanProcessor interface {
    Process(input Message) error
    io.Closer
}

// Consumer 客户端应该实现的接口
type Consumer interface {
    Partitions() <-chan cluster.PartitionConsumer
    MarkPartitionOffset(topic string, partition int32, offset int64, metadata string)
    io.Closer
}

// ProcessorFactory 处理工厂
type ProcessorFactory struct {
    topic          string
    consumer       consumer.Consumer
    baseProcessor  processor.SpanProcessor
    // ...
}

// ingester 消费者，消费kafka 中的数据，并且将span 写入底层存储
type Consumer struct {
    internalConsumer consumer.Consumer // 内部队列消费者实例
    processorFactory ProcessorFactory // 处理工厂
    partitionIDToState  map[int32]*consumerState // 分片处理状态
}

type startedProcessor struct {
    services  []service // 控制kafka 消费偏移量
    processor startProcessor // 消息处理器，有多重中间件嵌套
}
```

创建数据存储实例：

```go
// 数据写入器
storageFactory, err := storage.NewFactory(storage.FactoryConfigFromEnvAndCLI(os.Args, os.Stderr))
spanWriter, err := storageFactory.CreateSpanWriter()
```

创建消费者实例：

```go
// github.com/jaegertracing/jaeger/cmd/ingester/app/builder/builder.go
// 创建span 消费者实例CreateConsumer creates a new span consumer for the ingester
func CreateConsumer(logger *zap.Logger, metricsFactory metrics.Factory, spanWriter spanstore.Writer, options app.Options) (*consumer.Consumer, error) {
    // 数据处理参数，包括将span 写入存储的写入实例 spanWriter
    spParams := processor.SpanProcessorParams{
        Writer:       spanWriter,
        Unmarshaller: unmarshaller,
    }
    
    // 创建span 处理实例，现在只提供消费kafka 的处理器
    spanProcessor := processor.NewSpanProcessor(spParams)

    // 创建kafka 消费者客户端
    saramaConsumer, err := consumerConfig.NewConsumer()
    
    factoryParams := consumer.ProcessorFactoryParams{
        SaramaConsumer: saramaConsumer, // 队列消费者客户端
        BaseProcessor:  spanProcessor, // 数据处理器
        // ...
    }
    // 创建处理工厂，包含队列消费者实例和数据处理器
    processorFactory, err := consumer.NewProcessorFactory(factoryParams)
        
    consumerParams := consumer.Params{
        InternalConsumer:      saramaConsumer,  // kafka 消费端实例
        ProcessorFactory:      *processorFactory, // 处理工厂
        // ...
    }
    // 创建ingester 实例
    return consumer.New(consumerParams)
}
```

### 1.1 processor 消息处理器

当前支持kafka 处理器。

```go
// github.com/jaegertracing/jaeger/cmd/ingester/app/processor/span_processor.go
// 创建消费Kafka 队列的span 处理器
func NewSpanProcessor(params SpanProcessorParams) *KafkaSpanProcessor {
    return &KafkaSpanProcessor{
        unmarshaller: params.Unmarshaller,
        writer:       params.Writer,
    }
}

// Process 处理kafka 队列中的span 数据，写入底层存储
func (s KafkaSpanProcessor) Process(message Message) error {
    mSpan, err := s.unmarshaller.Unmarshal(message.Value())
    return s.writer.WriteSpan(mSpan)
}
```

### 1.2 创建kafka 客户端实例

创建kafka 客户端实例，用来获取队列数据。

```go
// 创建kafka 消费者实例
func (c *Configuration) NewConsumer() (Consumer, error) {
    saramaConfig := cluster.NewConfig()
    saramaConfig.Group.Mode = cluster.ConsumerModePartitions
    saramaConfig.ClientID = c.ClientID
    c.AuthenticationConfig.SetConfiguration(&saramaConfig.Config)
    return cluster.NewConsumer(c.Brokers, c.GroupID, []string{c.Topic}, saramaConfig)
}
```

### 1.3 消息处理逻辑

```go
// github.com/jaegertracing/jaeger/cmd/ingester/app/consumer/processor_factory.go
// 创建一个处理工厂
func NewProcessorFactory(params ProcessorFactoryParams) (*ProcessorFactory, error) {
    return &ProcessorFactory{
        consumer:       params.SaramaConsumer, // 队列消费者实例
        baseProcessor:  params.BaseProcessor, // 数据处理器
        // ...
    }, nil
}
```


### 1.4 创建consumer 实例

```go
// github.com/jaegertracing/jaeger/cmd/ingester/app/consumer/consumer.go
// 创建消费者
func New(params Params) (*Consumer, error) {
    // ...
    return &Consumer{
        internalConsumer:    params.InternalConsumer,
        processorFactory:    params.ProcessorFactory,
        partitionIDToState:  make(map[int32]*consumerState),
        // ...
    }, nil
}
```


## 2. 开启消费逻辑

### 2.1 开启消费

```go
// github.com/jaegertracing/jaeger/cmd/ingester/app/consumer/consumer.go
// 开启consumer 实例，开始处理队列中的span 数据
func (c *Consumer) Start() {
    go func() {
        // Partitions() <-chan cluster.PartitionConsumer
        // 获取kafka 消费端实例的分区消费者实例
        for pc := range c.internalConsumer.Partitions() {
            // 获取一个分区消费者实例
            c.partitionIDToState[pc.Partition()] = &consumerState{partitionConsumer: pc}
            // 处理分区消费者实例中的队列数据
            go c.handleMessages(pc)
        }
    }()
}
```

### 2.2 handleMessages

处理分区消费者实例中的队列数据

```go
// github.com/jaegertracing/jaeger/cmd/ingester/app/consumer/consumer.go
// 通过消费者实例，获取消息队列中的span 数据，进行处理
func (c *Consumer) handleMessages(pc sc.PartitionConsumer) {
    // github.com/jaegertracing/jaeger/cmd/ingester/app/processor/span_processor.go
    // 对应processor 消息处理器，用来将消息写入 spanstore 中
    var msgProcessor processor.SpanProcessor

    // ...
    for {
        select {
        // 取出分区消费者实例中的队列消息
        case msg, ok := <-pc.Messages():
            // 获取processor 消息处理器实例
            msgProcessor = c.processorFactory.new(pc.Partition(), msg.Offset-1)
            defer msgProcessor.Close()
        
            // 处理数据，对应kafka 消息队里实例
            // 首先解析队列的span 数据，然后通过spanWriter 写入底层存储
            msgProcessor.Process(&saramaMessageWrapper{msg})
            // ...
        }
    }
}
```

### 2.3 为消息处理器增加中间件

中间件包括：

1. 重试
2. 提交
3. 记录处理延迟
4. 并发处理

```go
// github.com/jaegertracing/jaeger/cmd/ingester/app/consumer/processor_factory.go
// 获取processor 消息处理器实例
func (c *ProcessorFactory) new(partition int32, minOffset int64) processor.SpanProcessor {
    // 重试处理
    retryProcessor := decorator.NewRetryingProcessor(c.metricsFactory, c.baseProcessor)

    // 处理完成提交处理
    cp := NewCommittingProcessor(retryProcessor, om)

    // 处理时长记录
    spanProcessor := processor.NewDecoratedProcessor(c.metricsFactory, cp)

    // 并发处理
    pp := processor.NewParallelProcessor(spanProcessor, c.parallelism, c.logger)

    return newStartedProcessor(pp, om)
}

// 返回的对象实现了 processor.SpanProcessor 接口
func newStartedProcessor(parallelProcessor startProcessor, services ...service) processor.SpanProcessor {
    s := &startedProcessor{
        services:  services,
        processor: parallelProcessor,
    }

    // 开启并发
    s.processor.Start()
    return s
}

// 所有处理器都需要实现 Process 函数
func (c *startedProcessor) Process(message processor.Message) error {
    return c.processor.Process(message)
}
```

## 3. Process 中间件

实现了Process 接口的中间件。

### 3.1 重试

多次重试处理。

```go
// github.com/jaegertracing/jaeger/cmd/ingester/app/processor/decorator/retry.go
// retryProcessor := decorator.NewRetryingProcessor(c.metricsFactory, c.baseProcessor)
// 处理Process 处理失败，会进行多次尝试
func NewRetryingProcessor(f metrics.Factory, processor processor.SpanProcessor, opts ...RetryOption) processor.SpanProcessor {
    // ...
    return &retryDecorator{
        retryAttempts: m.Counter(metrics.Options{Name: "retry-attempts", Tags: nil}),
        exhausted:     m.Counter(metrics.Options{Name: "retry-exhausted", Tags: nil}),
        processor:     processor,
        options:       options,
    }
}

func (d *retryDecorator) Process(message processor.Message) error {
    // 第一次处理
    err := d.processor.Process(message)
    // 第一次处理成功则直接返回
    if err == nil {
        return nil
    }

    // 第一次没有处理成功会多次尝试
    for attempts := uint(0); err != nil && d.options.maxAttempts > attempts; attempts++ {
        time.Sleep(d.computeInterval(attempts))
        err = d.processor.Process(message)
        d.retryAttempts.Inc(1)
    }
    // ...
    return nil
}
```

### 3.2 提交

消息处理成功后增加kafka 偏移量。

```go
// github.com/jaegertracing/jaeger/cmd/ingester/app/consumer/committing_processor.go
// cp := NewCommittingProcessor(retryProcessor, om)
func NewCommittingProcessor(processor processor.SpanProcessor, marker offsetMarker) processor.SpanProcessor {
    return &comittingProcessor{
        processor: processor,
        marker:    marker,
    }
}

func (d *comittingProcessor) Process(message processor.Message) error {
    if msg, ok := message.(Message); ok {
        err := d.processor.Process(message)
        if err == nil {
            // 消息处理成功后，增加偏移量
            d.marker.MarkOffset(msg.Offset())
        }
        return err
    }
    return errors.New("committing processor used with non-kafka message")
}
```

### 3.3 处理耗时记录

```go
// github.com/jaegertracing/jaeger/cmd/ingester/app/processor/metrics_decorator.go
// spanProcessor := processor.NewDecoratedProcessor(c.metricsFactory, cp)
func NewDecoratedProcessor(f metrics.Factory, processor SpanProcessor) SpanProcessor {
    m := f.Namespace(metrics.NSOptions{Name: "span-processor", Tags: nil})
    return &metricsDecorator{
        errors:    m.Counter(metrics.Options{Name: "errors", Tags: nil}),
        latency:   m.Timer(metrics.TimerOptions{Name: "latency", Tags: nil}),
        processor: processor,
    }
}

func (d *metricsDecorator) Process(message Message) error {
    now := time.Now()

    // 记录处理延迟
    err := d.processor.Process(message)
    d.latency.Record(time.Since(now))
    // ...
    return err
}
```

### 3.4 并发处理

开启多个goroutine 并发处理消息，加快消息处理速度。

```go
// github.com/jaegertracing/jaeger/cmd/ingester/app/processor/parallel_processor.go
// pp := processor.NewParallelProcessor(spanProcessor, c.parallelism, c.logger)
func NewParallelProcessor(processor SpanProcessor, parallelism int, logger *zap.Logger) *ParallelProcessor {
    return &ParallelProcessor{
        messages:    make(chan Message), // 消息channel
        processor:   processor,
        numRoutines: parallelism,
        closed:      make(chan struct{}),
    }
}

// Start 开启goroutine，基于channel 消费消息
func (k *ParallelProcessor) Start() {
    for i := 0; i < k.numRoutines; i++ {
        k.wg.Add(1)
        go func() {
            for {
                select {
                case msg := <-k.messages:
                    k.processor.Process(msg)
                case <-k.closed:
                    k.wg.Done()
                    return
                }
            }
        }()
    }
}

// 将消息放入队列，有多个goroutine 进行快速消费
func (k *ParallelProcessor) Process(message Message) error {
    k.messages <- message
    return nil
}
```


## 参考资料

- github.com/jaegertracing/jaeger/cmd/ingester/main.go

