<!-- ---
title: Go-Micro grpc 服务实现
date: 2020-08-23 20:52:27
category: showcode, micro, go-micro
--- -->

# Go-Micro grpc 服务实现

使用grpc 创建服务。

![](images/go-micro_services_grpc.svg)

主要代码逻辑：

```go
// 创建grpc 服务实例
var DefaultServer server.Server = grpc.NewServer()

// 创建与注册处理handler
NewHandler()
Handle()

// 开启grpc 服务
Start()
```

主要数据结构：

```go
type grpcServer struct {
    rpc  *rServer
    srv  *grpc.Server
    opts        server.Options
    handlers    map[string]server.Handler
    subscribers map[*subscriber][]broker.Subscriber
    rsvc *registry.Service
}

type Options struct {
    Codecs       map[string]codec.NewCodec
    Broker       broker.Broker
    Registry     registry.Registry
    Router Router
}

type rpcHandler struct {
    name      string
    handler   interface{}
    endpoints []*registry.Endpoint
    opts      server.HandlerOptions
}

type service struct {
    name   string
    rcvr   reflect.Value
    typ    reflect.Type
    // 注册的方法
    method map[string]*methodType
}

type rServer struct {
    serviceMap map[string]*service
}
```

## 1. 创建grpc 服务实例

```go
// 创建grpc 服务实例
var DefaultServer server.Server = grpc.NewServer()

func NewServer(opts ...server.Option) server.Server {
    return newGRPCServer(opts...)
}

func newGRPCServer(opts ...server.Option) server.Server {
    options := newOptions(opts...)

    // 创建grpc 服务
    srv := &grpcServer{
        opts: options,
        rpc: &rServer{
            serviceMap: make(map[string]*service),
        },
        handlers:    make(map[string]server.Handler),
        subscribers: make(map[*subscriber][]broker.Subscriber),
    }

    // grpc 服务配置
    srv.configure()

    return srv
}

func newOptions(opt ...server.Option) server.Options {
    opts := server.Options{
        Broker:           http.NewBroker(),
        Registry:         mdns.NewRegistry(),
    }

    for _, o := range opt {
        o(&opts)
    }

    return opts
}

func (g *grpcServer) configure(opts ...server.Option) {
    // ...
    for _, o := range opts {
        o(&g.opts)
    }

    gopts := []grpc.ServerOption{
        grpc.MaxRecvMsgSize(maxMsgSize),
        grpc.MaxSendMsgSize(maxMsgSize),
        // 注册处理逻辑
        grpc.UnknownServiceHandler(g.handler),
    }

    // 创建grpc 服务
    g.srv = grpc.NewServer(gopts...)
}
```

## 2. 创建与注册处理handler

```go
// 创建处理handler
NewHandler()

func (g *grpcServer) NewHandler(h interface{}, opts ...server.HandlerOption) server.Handler {
    return newRpcHandler(h, opts...)
}

func newRpcHandler(handler interface{}, opts ...server.HandlerOption) server.Handler {
    // ...
    typ := reflect.TypeOf(handler)
    hdlr := reflect.ValueOf(handler)
    name := reflect.Indirect(hdlr).Type().Name()

    var endpoints []*registry.Endpoint
    for m := 0; m < typ.NumMethod(); m++ {
        if e := extractEndpoint(typ.Method(m)); e != nil {
            e.Name = name + "." + e.Name
            endpoints = append(endpoints, e)
        }
    }

    return &rpcHandler{
        name:      name,
        handler:   handler,
        endpoints: endpoints,
        opts:      options,
    }
}
```

```go
// 注册处理handler
Handle()

func (g *grpcServer) Handle(h server.Handler) error {
    g.rpc.register(h.Handler())

    g.handlers[h.Name()] = h
    return nil
}

func (server *rServer) register(rcvr interface{}) error {
    s := new(service)
    s.typ = reflect.TypeOf(rcvr)
    s.rcvr = reflect.ValueOf(rcvr)
    sname := reflect.Indirect(s.rcvr).Type().Name()
    
    // ...
    s.name = sname
    s.method = make(map[string]*methodType)

    // 遍历所有导出方法
    for m := 0; m < s.typ.NumMethod(); m++ {
        method := s.typ.Method(m)
        if mt := prepareEndpoint(method); mt != nil {
            s.method[method.Name] = mt
        }
    }

    // 注册的handler
    server.serviceMap[s.name] = s
    return nil
}
```

## 3. grpc 请求处理

基于 grpc 的UnknownServiceHandler 配置，接管grpc handler 注册和路由逻辑。

```go
grpc.UnknownServiceHandler(g.handler)

// 处理请求路由逻辑
func (g *grpcServer) handler(srv interface{}, stream grpc.ServerStream) (err error) {
    // 获取请求路由信息
    fullMethod, ok := grpc.MethodFromServerStream(stream)

    serviceName, methodName, err := mgrpc.ServiceMethod(fullMethod)
    
    // 获取处理handler
    service := g.rpc.serviceMap[serviceName]
    
    // 请求处理
    return g.processRequest(stream, service, mtype, ct, ctx)
}

func (g *grpcServer) processRequest(stream grpc.ServerStream, service *service, mtype *methodType, ct string, ctx context.Context) error {
    for {
        // 接收请求信息
        stream.RecvMsg(argv.Interface())

        // 解码请求信息
        cc, err := g.newGRPCCodec(ct)
        b, err := cc.Marshal(argv.Interface())

        // 请求参数
        r := &rpcRequest{
            service:     g.opts.Name,
            contentType: ct,
            method:      fmt.Sprintf("%s.%s", service.name, mtype.method.Name),
            body:        b,
            payload:     argv.Interface(),
        }

        // 请求处理
        fn := func(ctx context.Context, req server.Request, rsp interface{}) (err error) {
            returnValues = function.Call([]reflect.Value{service.rcvr, mtype.prepareContext(ctx), reflect.ValueOf(argv.Interface()), reflect.ValueOf(rsp)})

            return err
        }

        // 处理中间件
        for i := len(g.opts.HdlrWrappers); i > 0; i-- {
            fn = g.opts.HdlrWrappers[i-1](fn)
        }

        // 执行请求
        fn(ctx, r, replyv.Interface())

        // 返回处理结果
        stream.SendMsg(replyv.Interface())
    }
}
```

## 4. 开启grpc 服务

```go
// 开启grpc 服务
Start()

func (g *grpcServer) Start() error {
    // 网络监听
    ts, err = net.Listen("tcp", config.Address)

    // 订阅监听
    if len(g.subscribers) > 0 {
        // ...
        config.Broker.Connect()
    }

    // 服务注册，用于服务发现
    g.Register()

    // 服务监听处理
    go func() {
        // grpc 服务监听启动
        g.srv.Serve(ts)
    }()

    go func() {
        for {
            select {
            // 定期注册服务
            case <-t.C:
                g.Register()
            }
        }
    }()

    return nil
}
```

服务注册：

```go
func (g *grpcServer) Register() error {
    config := g.opts

    regFunc := func(service *registry.Service) error {
        for i := 0; i < 3; i++ {
            // 服务注册
            config.Registry.Register(service, rOpts...)
        }
    }

    // register 的服务节点
    node := &registry.Node{
        Id:       config.Name + "-" + config.Id,
        Address:  mnet.HostPort(addr, port),
        Metadata: md,
    }

    // 注册服务端点
    for n, e := range g.handlers {
        handlerList = append(handlerList, n)
    }
    for e := range g.subscribers {
        subscriberList = append(subscriberList, e)
    }

    for _, n := range handlerList {
        endpoints = append(endpoints, g.handlers[n].Endpoints()...)
    }
    for _, e := range subscriberList {
        endpoints = append(endpoints, e.Endpoints()...)
    }

    service := &registry.Service{
        Name:      config.Name,
        Version:   config.Version,
        Nodes:     []*registry.Node{node},
        Endpoints: endpoints,
    }

    // 注册服务
    regFunc(service)

    return nil
}
```

## 参考资料

- github.com/micro/go-micro/server/grpc/grpc.go
