<!-- ---
title: grpc client transport
date: 2020-10-31 09:17:38
category: showcode, grpc
--- -->

# Grpc 客户端传输层实现

主要逻辑：

```go
// 创建传输层
transport.NewClientTransport(connectCtx, ac.cc.ctx, addr, copts, onPrefaceReceipt, onGoAway, onClose)

// 创建连接
a.t.NewStream(cs.ctx, cs.callHdr)

// 发送请求
a.t.Write(a.s, hdr, payld, &transport.Options{Last: !cs.desc.ClientStreams})

// 读取响应
p.r.Read(msg)
```

![](images/grpc_client_balancer.svg)

主要数据结构：

```go
type http2Client struct {
    conn       net.Conn
    loopy      *loopyWriter
    framer *framer
    controlBuf *controlBuffer
}
```

## 1. 创建http2 处理

```go
transport.NewClientTransport(connectCtx, ac.cc.ctx, addr, copts, onPrefaceReceipt, onGoAway, onClose)
```

```go
func NewClientTransport(connectCtx, ctx context.Context, addr resolver.Address, opts ConnectOptions, onPrefaceReceipt func(), onGoAway func(GoAwayReason), onClose func()) (ClientTransport, error) {
    return newHTTP2Client(connectCtx, ctx, addr, opts, onPrefaceReceipt, onGoAway, onClose)
}

func newHTTP2Client(connectCtx, ctx context.Context, addr resolver.Address, opts ConnectOptions, onPrefaceReceipt func(), onGoAway func(GoAwayReason), onClose func()) (_ *http2Client, err error) {
    // ...
    conn, err := dial(connectCtx, opts.Dialer, addr, opts.UseProxy, opts.UserAgent)

    // 创建http2 客户端
    t := &http2Client{
        md:                    addr.Metadata,
        conn:                  conn,
        activeStreams:         make(map[uint32]*Stream),
    }

    // 异步处理数据帧缓冲
    t.controlBuf = newControlBuffer(t.ctxDone)

    // 读取数据帧
    go t.reader()

    go func() {
        // 数据帧发送异步处理
        t.loopy = newLoopyWriter(clientSide, t.framer, t.controlBuf, t.bdpEst)
        err := t.loopy.run()
        // ...
    }()
    return t, nil
}
```

建立网络连接。

```go
func dial(ctx context.Context, fn func(context.Context, string) (net.Conn, error), addr resolver.Address, useProxy bool, grpcUA string) (net.Conn, error) {
    // ...
    return (&net.Dialer{}).DialContext(ctx, networkType, addr.Addr)
}
```

## 2. 创建数据流

```go
func (t *http2Client) NewStream(ctx context.Context, callHdr *CallHdr) (_ *Stream, err error) {
    headerFields, err := t.createHeaderFields(ctx, callHdr)
    
    // ...
    s := t.newStream(ctx, callHdr)
    
    hdr := &headerFrame{
        hf:        headerFields,
    }

    // 发送控制帧
    success, err := t.controlBuf.executeAndPut(..., hdr)
    
    // ...
    return s, nil
}
```

### 2.1 请求头信息

```go
func (t *http2Client) createHeaderFields(ctx context.Context, callHdr *CallHdr) ([]hpack.HeaderField, error) {
    // ...
    headerFields := make([]hpack.HeaderField, 0, hfLen)
    headerFields = append(headerFields, hpack.HeaderField{Name: ":method", Value: "POST"})
    headerFields = append(headerFields, hpack.HeaderField{Name: ":scheme", Value: t.scheme})
    headerFields = append(headerFields, hpack.HeaderField{Name: ":path", Value: callHdr.Method})

    return headerFields, nil
}
```

### 2.2 创建请求流

```go
func (t *http2Client) newStream(ctx context.Context, callHdr *CallHdr) *Stream {
    // 创建请求流
    s := &Stream{
        ct:             t,
        method:         callHdr.Method,
        buf:            newRecvBuffer(),
        contentSubtype: callHdr.ContentSubtype,
    }
    
    // ...
    s.trReader = &transportReader{
        reader: &recvBufferReader{
            ctx:     s.ctx,
            ctxDone: s.ctx.Done(),
            recv:    s.buf,
        },
    }
    return s
}
```

## 3. 请求数据读写

数据读写异步实现。

### 3.1 发送请求

```go
func (t *http2Client) Write(s *Stream, hdr []byte, data []byte, opts *Options) error {
    // ...
    df := &dataFrame{
        streamID:  s.id,
        endStream: opts.Last,
        h:         hdr,
        d:         data,
    }

    return t.controlBuf.put(df)
}
```

### 3.2 读取响应

异步读取Stream 数据。

```go
p.r.Read(msg)

func (s *Stream) Read(p []byte) (n int, err error) {
    // 读取数据
    return io.ReadFull(s.trReader, p)
}

func (t *transportReader) Read(p []byte) (n int, err error) {
    n, err = t.reader.Read(p)
}

func (r *recvBufferReader) Read(p []byte) (n int, err error) {
    // ...
    n, r.err = r.read(p)
    return n, r.err
}

func (r *recvBufferReader) read(p []byte) (n int, err error) {
    select {
    case m := <-r.recv.get():
        return r.readAdditional(m, p)
    }
}

func (r *recvBufferReader) readAdditional(m recvMsg, p []byte) (n int, err error) {
    // 读取数据
    copied, _ := m.buffer.Read(p)
    // ...
}
```

## 参考资料

- github.com/grpc/grpc-go/internal/transport/http2_client.go

