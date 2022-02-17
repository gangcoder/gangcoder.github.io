<!-- ---
title: Grpc 服务端服务实现
date: 2020-09-14 22:50:22
category: showcode, grpc
--- -->

# Grpc 服务端服务实现

创建Grpc 服务主要逻辑：

```go
// 监听端口
lis, err := net.Listen("tcp", port)

// 创建grpc 服务实例
s := grpc.NewServer()

// 注册grpc 服务
pb.RegisterGreeterService(s, &pb.GreeterService{SayHello: sayHello})

// 运行grpc 服务
s.Serve(lis)
```

![](images/grpc_server.svg)

主要数据结构：

```go
type Server struct {
    lis      map[net.Listener]bool
    conns    map[transport.ServerTransport]bool
    services map[string]*serviceInfo
}
```

## 1. 创建grpc 服务实例

```go
s := grpc.NewServer()
```

```go
func NewServer(opt ...ServerOption) *Server {
    opts := defaultServerOptions
    for _, o := range opt {
        o.apply(&opts)
    }
    s := &Server{
        lis:      make(map[net.Listener]bool),
        opts:     opts,
        conns:    make(map[transport.ServerTransport]bool),
        services: make(map[string]*serviceInfo),
        quit:     grpcsync.NewEvent(),
        done:     grpcsync.NewEvent(),
        czData:   new(channelzData),
    }

}
```

## 2. 注册grpc 服务

将grpc 服务注册到服务内部。

```go
pb.RegisterGreeterServer(s, &server{})

func RegisterGreeterServer(s grpc.ServiceRegistrar, srv GreeterServer) {
    s.RegisterService(&_Greeter_serviceDesc, srv)
}

var _Greeter_serviceDesc = grpc.ServiceDesc{
    ServiceName: "helloworld.Greeter",
    Methods: []grpc.MethodDesc{
        {
            MethodName: "SayHello",
            Handler:    _Greeter_SayHello_Handler,
        },
    },
}

// 注册服务
func (s *Server) RegisterService(sd *ServiceDesc, ss interface{}) {
    // ...
    s.register(sd, ss)
}

// 将服务信息，写入 services
func (s *Server) register(sd *ServiceDesc, ss interface{}) {
    // ...
    info := &serviceInfo{
        serviceImpl: ss,
        methods:     make(map[string]*MethodDesc),
    }
    for i := range sd.Methods {
        d := &sd.Methods[i]
        info.methods[d.MethodName] = d
    }

    s.services[sd.ServiceName] = info
}
```

## 3. 运行grpc 服务

```go
s.Serve(lis)

func (s *Server) Serve(lis net.Listener) error {
    // ...
    ls := &listenSocket{Listener: lis}
    s.lis[ls] = true
    
    for {
        // 接收请求，处理请求
        rawConn, err := lis.Accept()
        // ...
        go func() {
            s.handleRawConn(rawConn)
        }()
    }
}
```

处理请求。

```go
func (s *Server) handleRawConn(rawConn net.Conn) {
    // 创建http2 传输处理层
    st := s.newHTTP2Transport(conn, authInfo)
    
    // 处理请求
    go func() {
        s.serveStreams(st)
    }()
}

// 处理请求流
func (s *Server) serveStreams(st transport.ServerTransport) {
    // ...
    st.HandleStreams(func(stream *transport.Stream) {
        // ...
        go func() {
            // 处理请求
            s.handleStream(st, stream, s.traceInfo(st, stream))
        }()
    }, ...)
}
```

处理一条请求流。

```go
func (s *Server) handleStream(t transport.ServerTransport, stream *transport.Stream, trInfo *traceInfo) {
    sm := stream.Method()
    
    // ...
    service := sm[:pos]
    method := sm[pos+1:]

    // 获取服务
    srv, knownService := s.services[service]
    if knownService {
        // 获取请求的方法
        if md, ok := srv.methods[method]; ok {
            s.processUnaryRPC(t, stream, srv, md, trInfo)
            return
        }
        if sd, ok := srv.streams[method]; ok {
            s.processStreamingRPC(t, stream, srv, sd, trInfo)
            return
        }
    }
    // ...
}
```

## 4. 一元请求处理

一元请求，处理一元请求。

```go
s.processUnaryRPC(t, stream, srv, md, trInfo)
```

```go
func (s *Server) processUnaryRPC(t transport.ServerTransport, stream *transport.Stream, info *serviceInfo, md *MethodDesc, trInfo *traceInfo) (err error) {
    // ...

    // 接收请求数据
    d, err := recvAndDecompress(&parser{r: stream}, stream, dc, s.opts.maxReceiveMessageSize, payInfo, decomp)

    // 解析数据
    df := func(v interface{}) error {
        s.getCodec(stream.ContentSubtype()).Unmarshal(d, v)
        
        return nil
    }
    
    // 处理请求
    reply, appErr := md.Handler(info.serviceImpl, ctx, df, s.opts.unaryInt)

    // ...

    // 返回响应数据
    s.sendResponse(t, stream, reply, cp, opts, comp)

    // 返回响应状态
    err = t.WriteStatus(stream, statusOK)
}
```

### 4.1 接收请求数据

```go
recvAndDecompress(&parser{r: stream}, stream, dc, s.opts.maxReceiveMessageSize, payInfo, decomp)

func recvAndDecompress(p *parser, s *transport.Stream, dc Decompressor, maxReceiveMessageSize int, payInfo *payloadInfo, compressor encoding.Compressor) ([]byte, error) {
    // 接收数据
    pf, d, err := p.recvMsg(maxReceiveMessageSize)
    // 解压数据
    d, size, err = decompress(compressor, d, maxReceiveMessageSize)

    // ...
    return d, nil
}

func (p *parser) recvMsg(maxReceiveMessageSize int) (pf payloadFormat, msg []byte, err error) {
    // ...
    // 读取头信息
    p.r.Read(p.header[:])

    // 解析头信息
    pf = payloadFormat(p.header[0])
    
    // 读取内容
    p.r.Read(msg)
    return pf, msg, nil
}
```

### 4.2 数据内容解码

```go
s.getCodec(stream.ContentSubtype()).Unmarshal(d, v)

// 请求内容子类型，选定解码器
func (s *Server) getCodec(contentSubtype string) baseCodec {
    // 获取编码器
    codec := encoding.GetCodec(contentSubtype)

    // ...
    return codec
}
```

### 4.3 请求处理

```go
// 取出注册的服务，调用handler 处理请求
// handler 对应 _Greeter_SayHello_Handler
md.Handler(info.serviceImpl, ctx, df, s.opts.unaryInt)

// 请求处理
func _Greeter_SayHello_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
    in := new(HelloRequest)
    // 解析请求数据
    if err := dec(in); err != nil {
        return nil, err
    }
    
    // 处理请求
    return srv.(GreeterServer).SayHello(ctx, in)
}
```

### 4.4 响应结果

```go
s.sendResponse(t, stream, reply, cp, opts, comp)

func (s *Server) sendResponse(t transport.ServerTransport, stream *transport.Stream, msg interface{}, cp Compressor, opts *transport.Options, comp encoding.Compressor) error {
    // 数据编码
    data, err := encode(s.getCodec(stream.ContentSubtype()), msg)
    
    // 数据压缩
    compData, err := compress(data, cp, comp)
    
    // 响应头信息
    hdr, payload := msgHeader(data, compData)
    
    // 写入响应数据
    err = t.Write(stream, hdr, payload, opts)
    
    // ...
    return err
}
```

## 参考资料

- github.com/grpc/grpc-go/server.go

