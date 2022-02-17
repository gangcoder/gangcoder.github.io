<!-- ---
title: Grpc 客户端调用实现
date: 2020-09-14 22:50:10
category: showcode, grpc
--- -->

# Grpc 客户端调用实现

grpc 客户端调用主要逻辑：

```go
// 开启网络连接
conn, err := grpc.Dial(address, grpc.WithInsecure(), grpc.WithBlock())

// 创建客户端实例
c := pb.NewGreeterClient(conn)

// 调用客户端方法
r, err := c.SayHello(ctx, &pb.HelloRequest{Name: name})
```

![](images/grpc_client.svg)

主要数据结构：

```go
// 解析Builder
type Builder interface {
    Build(target Target, cc ClientConn, opts BuildOptions) (Resolver, error)
    Scheme() string
}

// 解析器，监控地址变更
type Resolver interface {
    ResolveNow(ResolveNowOptions)
    Close()
}

type ccResolverWrapper struct {
    cc         *ClientConn
    resolver   resolver.Resolver
    curState   resolver.State
}

type greeterClient struct {
    cc grpc.ClientConnInterface
}

type ClientConnInterface interface {
    Invoke(ctx context.Context, method string, args interface{}, reply interface{}, opts ...CallOption) error
    NewStream(ctx context.Context, desc *StreamDesc, method string, opts ...CallOption) (ClientStream, error)
}
```

## 1. 开启网络连接

```go
conn, err := grpc.Dial(address, grpc.WithInsecure(), grpc.WithBlock())

func Dial(target string, opts ...DialOption) (*ClientConn, error) {
    return DialContext(context.Background(), target, opts...)
}

func DialContext(ctx context.Context, target string, opts ...DialOption) (conn *ClientConn, err error) {
    cc := &ClientConn{
        target:            target,
        csMgr:             &connectivityStateManager{},
        conns:             make(map[*addrConn]struct{}),
        dopts:             defaultDialOptions(),
        blockingpicker:    newPickerWrapper(),
    }
    
    for _, opt := range opts {
        opt.apply(&cc.dopts)
    }
    
    // 地址解析器实现
    cc.parsedTarget = grpcutil.ParseTarget(cc.target, cc.dopts.copts.Dialer != nil)
    resolverBuilder := cc.getResolver(cc.parsedTarget.Scheme)
    // ...
    rWrapper, err := newCCResolverWrapper(cc, resolverBuilder)
    cc.resolverWrapper = rWrapper

    return cc, nil
}
```

### 1.1 获取解析构建器

根据地址schema 获取解析器具体实现。

主要解析器实现：
1. passthrough，默认项
2. dns

```go
resolverBuilder := cc.getResolver(cc.parsedTarget.Scheme)

func (cc *ClientConn) getResolver(scheme string) resolver.Builder {
    // ...
    return resolver.Get(scheme)
}

func Get(scheme string) Builder {
    if b, ok := m[scheme]; ok {
        return b
    }
    return nil
}
```

### 1.2 创建解析器

```go
rWrapper, err := newCCResolverWrapper(cc, resolverBuilder)

func newCCResolverWrapper(cc *ClientConn, rb resolver.Builder) (*ccResolverWrapper, error) {
    ccr := &ccResolverWrapper{
        cc:   cc,
    }

    // ...
    ccr.resolver, err = rb.Build(cc.parsedTarget, ccr, rbo)
    return ccr, nil
}
```

## 2. 创建客户端实例

```go
c := pb.NewGreeterClient(conn)

func NewGreeterClient(cc grpc.ClientConnInterface) GreeterClient {
    return &greeterClient{cc}
}
```

## 3. 调用客户端方法

```go
r, err := c.SayHello(ctx, &pb.HelloRequest{Name: name})

func (c *greeterClient) SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloReply, error) {
    out := new(HelloReply)
    err := c.cc.Invoke(ctx, "/helloworld.Greeter/SayHello", in, out, opts...)

    return out, nil
}
```

```go
func (cc *ClientConn) Invoke(ctx context.Context, method string, args, reply interface{}, opts ...CallOption) error {
    // ...
    opts = combine(cc.dopts.callOptions, opts)

    // 中间件
    if cc.dopts.unaryInt != nil {
        return cc.dopts.unaryInt(ctx, method, args, reply, cc, invoke, opts...)
    }
    return invoke(ctx, method, args, reply, cc, opts...)
}

func invoke(ctx context.Context, method string, req, reply interface{}, cc *ClientConn, opts ...CallOption) error {
    // 创建流
    cs, err := newClientStream(ctx, unaryStreamDesc, cc, method, opts...)
    
    // ...
    // 发送请求
    if err := cs.SendMsg(req); err != nil {
        return err
    }
    
    // 接收响应
    return cs.RecvMsg(reply)
}
```

### 3.1 创建请求流

```go
func newClientStream(ctx context.Context, desc *StreamDesc, cc *ClientConn, method string, opts ...CallOption) (_ ClientStream, err error) {
    // ...
    callHdr := &transport.CallHdr{
        Host:           cc.authority,
        Method:         method,
        ContentSubtype: c.contentSubtype,
    }

    cs := &clientStream{
        callHdr:      callHdr,
        ctx:          ctx,
        methodConfig: &mc,
        opts:         opts,
        callInfo:     c,
        cc:           cc,
        desc:         desc,
        codec:        c.codec,
    }

    // 创建重试器
    cs.newAttemptLocked(sh, trInfo)

    op := func(a *csAttempt) error { return a.newStream() }
    cs.withRetry(op, func() { cs.bufferForRetryLocked(0, op) })

    return cs, nil
}
```

```go
cs.newAttemptLocked(sh, trInfo)

func (cs *clientStream) newAttemptLocked(sh stats.Handler, trInfo *traceInfo) (retErr error) {
    newAttempt := &csAttempt{
        cs:           cs,
        dc:           cs.cc.dopts.dc,
        statsHandler: sh,
        trInfo:       trInfo,
    }
    
    // ...

    // 获取传输层
    t, done, err := cs.cc.getTransport(ctx, cs.callInfo.failFast, cs.callHdr.Method)

    newAttempt.t = t
    newAttempt.done = done
    cs.attempt = newAttempt
    return nil
}
```

获取传输层。

```go
func (cc *ClientConn) getTransport(ctx context.Context, failfast bool, method string) (transport.ClientTransport, func(balancer.DoneInfo), error) {
    t, done, err := cc.blockingpicker.pick(ctx, ...)
    // ...
    return t, done, nil
}
```

创建请求流。

```go
// a.newStream()
func (a *csAttempt) newStream() error {
    cs := a.cs
    s, err := a.t.NewStream(cs.ctx, cs.callHdr)
    // ...

    cs.attempt.s = s
    cs.attempt.p = &parser{r: s}
    return nil
}
```

### 3.2 发送请求

```go
func (cs *clientStream) SendMsg(m interface{}) (err error) {
    // 预处理数据
    hdr, payload, data, err := prepareMsg(m, cs.codec, cs.cp, cs.comp)

    // 发送消息
    err := a.sendMsg(m, hdr, payload, data)
}

func prepareMsg(m interface{}, codec baseCodec, cp Compressor, comp encoding.Compressor) (hdr, payload, data []byte, err error) {
    // 序列化
    data, err = encode(codec, m)
    
    // 压缩
    compData, err := compress(data, cp, comp)
    
    // 头信息
    hdr, payload = msgHeader(data, compData)
    return hdr, payload, data, nil
}

func (a *csAttempt) sendMsg(m interface{}, hdr, payld, data []byte) error {
    cs := a.cs
    
    // ...
    a.t.Write(a.s, hdr, payld, &transport.Options{Last: !cs.desc.ClientStreams})
    return nil
}
```

### 3.3 接收响应

```go
return cs.RecvMsg(reply)

func (cs *clientStream) RecvMsg(m interface{}) error {
    // ...
    err := cs.withRetry(func(a *csAttempt) error {
        return a.recvMsg(m, recvInfo)
    }, cs.commitAttemptLocked)

    return err
}

func (a *csAttempt) recvMsg(m interface{}, payInfo *payloadInfo) (err error) {
    // ...
    err = recv(a.p, cs.codec, a.s, a.dc, m, *cs.callInfo.maxReceiveMessageSize, payInfo, a.decomp)

    d, err := recvAndDecompress(p, s, dc, maxReceiveMessageSize, payInfo, compressor)

    err := c.Unmarshal(d, m)
}
```

## 参考资料

- github.com/grpc/grpc-go/call.go