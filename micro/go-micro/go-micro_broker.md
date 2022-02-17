<!-- ---
title: Go-Micro 事件订阅与消费实现
date: 2020-08-23 20:52:50
category: showcode, micro, go-micro
--- -->

# Go-Micro 事件订阅与消费实现

主要调用逻辑：

```go
// 订阅消息
service.Subscribe("messages", Sub)

// 创建消息
ev := service.NewEvent("messages")

// 发布消息
ev.Publish(...)
```

![](images/go-micro_broker.svg)

主要数据结构：

```go
type Subscriber interface {
    Topic() string
    Subscriber() interface{}
    Endpoints() []*registry.Endpoint
    Options() SubscriberOptions
}

// nats 消息订阅
type natsBroker struct {
    addrs []string
    conn  *nats.Conn
    opts  broker.Options
    nopts nats.Options
}
```

## 1. 订阅消息

订阅消息实现

```go
// Subscribe 注册订阅器
func (s *Service) Subscribe(topic string, v interface{}) error {
    return s.Server().Subscribe(s.Server().NewSubscriber(topic, v))
}
```

### 1.1 创建订阅者实例

```go
// 创建订阅实例
s.Server().NewSubscriber(topic, v)

// 创建grpc 订阅实例
func (g *grpcServer) NewSubscriber(topic string, sb interface{}, opts ...server.SubscriberOption) server.Subscriber {
    return newSubscriber(topic, sb, opts...)
}

// 创建订阅处理实例
func newSubscriber(topic string, sub interface{}, opts ...server.SubscriberOption) server.Subscriber {
    var handlers []*handler
    if typ := reflect.TypeOf(sub); typ.Kind() == reflect.Func {
        h := &handler{
            method: reflect.ValueOf(sub),
        }

        handlers = append(handlers, h)

        // 注册端点信息
        endpoints = append(endpoints, &registry.Endpoint{
            Name:    "Func",
            Request: extractSubValue(typ),
            Metadata: map[string]string{
                "topic":      topic,
                "subscriber": "true",
            },
        })
    }

    return &subscriber{
        rcvr:       reflect.ValueOf(sub),
        typ:        reflect.TypeOf(sub),
        topic:      topic,
        subscriber: sub,
        handlers:   handlers,
        endpoints:  endpoints,
    }
}
```

### 1.2 注册订阅

```go
// 注册订阅实例
s.Server().Subscribe(...)

func (g *grpcServer) Subscribe(sb server.Subscriber) error {
    // 注册订阅实例
    g.subscribers[sub] = nil
    // ...
    return nil
}
```

## 2. 连接Broker

grpc 服务实现中，连接和注册订阅者实例到 broker。

启动服务时，如果有订阅器，还需要连接到broker。

```go
func (g *grpcServer) Start() error {
    // ...
    if len(g.subscribers) > 0 {
        // 连接到broker
        config.Broker.Connect()
    }

    return nil
}

// 注册订阅器到服务发现
func (g *grpcServer) Register() error {
    for sb := range g.subscribers {
        handler := g.createSubHandler(sb, g.opts)
        
        // 订阅消息
        sub, err := config.Broker.Subscribe(sb.Topic(), handler, opts...)
        
        // ...
        g.subscribers[sb] = []broker.Subscriber{sub}
    }

    return nil
}
```

创建消息处理handler。

```go
func (g *grpcServer) createSubHandler(sb *subscriber, opts server.Options) broker.Handler {
    return func(msg *broker.Message) (err error) {
        // ...
        for i := 0; i < len(sb.handlers); i++ {
            handler := sb.handlers[i]

            fn := func(ctx context.Context, msg server.Message) error {
                // 消息处理
                returnValues := handler.method.Call(vals)
            }

            // 中间件逻辑
            for i := len(opts.SubWrappers); i > 0; i-- {
                fn = opts.SubWrappers[i-1](fn)
            }

            go func() {
                // 消息处理
                err := fn(ctx, &rpcMessage{
                    topic:       sb.topic,
                    contentType: ct,
                    payload:     req.Interface(),
                    header:      msg.Header,
                    body:        msg.Body,
                })
                results <- err
            }()
        }

        return err
    }
}
```

## 3. 消息发布

创建消息体，连接到broker，再将消息发送到broker。

```go
// 发布消息
muclient.Publish(ctx, muclient.NewMessage(e.topic, msg), opts...)

// grpc 创建消息
func (g *grpcClient) NewMessage(topic string, msg interface{}, opts ...client.MessageOption) client.Message {
    return newGRPCEvent(topic, msg, g.opts.ContentType, opts...)
}

func newGRPCEvent(topic string, payload interface{}, contentType string, opts ...client.MessageOption) client.Message {
    // ...
    return &grpcEvent{
        payload:     payload,
        topic:       topic,
        contentType: contentType,
    }
}

// grpc 发布消息
func (g *grpcClient) Publish(ctx context.Context, p client.Message, opts ...client.PublishOption) error {
    // 连接到 Broker
    g.opts.Broker.Connect()

    // 发布消息的 topic
    topic := p.Topic()

    // 发布消息到 Broker
    return g.opts.Broker.Publish(topic, &broker.Message{
        Header: md,
        Body:   body,
    }, broker.PublishContext(options.Context))
}
```


## 4. Broker 实现

创建连接到 nats 的Broker。

```go
// 创建Broker 实例，通过配置指定
func NewBroker(opts ...broker.Option) broker.Broker {
    // ...
    n := &natsBroker{
        opts: options,
    }

    return n
}

// config.Broker.Connect()
// 连接到 nats
func (n *natsBroker) Connect() error {
    // ...
    switch status {
    case nats.CONNECTED, nats.RECONNECTING, nats.CONNECTING:
        n.connected = true
        return nil
    default: // DISCONNECTED or CLOSED or DRAINING
        opts := n.nopts
        opts.Servers = n.addrs
        
        // 创建连接
        c, err := opts.Connect()

        // ...
        n.conn = c
        n.connected = true
        return nil
    }
}
```

订阅消息实现：最终通过调用 nats 的订阅函数实现。

```go
// config.Broker.Subscribe(sb.Topic(), handler, opts...)

// 订阅消息处理
func (n *natsBroker) Subscribe(topic string, handler broker.Handler, opts ...broker.SubscribeOption) (broker.Subscriber, error) {
    // ...
    // 消息处理handler
    fn := func(msg *nats.Msg) {
        // 解码消息数据
        err := n.opts.Codec.Unmarshal(msg.Data, &m)
        // 处理消息
        handler(m)
    }

    // 订阅消息
    sub, err = n.conn.Subscribe(topic, fn)
    
    // 返回消息订阅者实例，用于取消订阅
    return &subscriber{s: sub, opts: opt}, nil
}
```

发布消息实现。

```go
// Broker.Publish
func (n *natsBroker) Publish(topic string, msg *broker.Message, opts ...broker.PublishOption) error {
    // 序列化消息数据
    b, err := n.opts.Codec.Marshal(msg)
    // 调用连接发布消息
    return n.conn.Publish(topic, b)
}
```

## 参考资料

- github.com/micro/go-micro/broker/broker.go
- github.com/micro/go-micro/broker/nats/nats.go
