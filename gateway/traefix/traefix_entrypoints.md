<!-- ---
title: Traefix EntryPoints
date: 2020-07-15 23:52:25
category: showcode, gateway, traefix
--- -->

# Traefix EntryPoints

Traefix 入口逻辑实现，主要逻辑：
1. 创建入口实例
   1. 接口监听
   2. HTTP 协议处理
   3. TCP 协议处理
2. 更新TCP 请求处理handler，这里基于tcp 协议的请求包括http 请求
3. 开启入口服务

![](images/traefix_entrypoints.svg)

主要代码：

```go
// 创建TCP 入口
serverEntryPointsTCP, err := server.NewTCPEntryPoints(staticConfiguration.EntryPoints)

// 更新请求处理 handler，这里包括http 请求和tcp 协议请求
serverEntryPointsTCP.Switch(routers)

// 开启入口服务
s.tcpEntryPoints.Start()
```

主要数据结构：

```go
// TCPEntryPoint TCP 入口结构
type TCPEntryPoint struct {
    listener net.Listener
    switcher *tcp.HandlerSwitcher // tcp 协议转发器
    httpServer *httpServer // http 请求处理handler
}

// HandlerSwitcher TCP 协议转发handler
type HandlerSwitcher struct {
    // 路由器，专门用于转发请求
    // 对应下面的 TCP 协议处理路由表
    router safe.Safe
}

// TCP 协议处理路由表
type Router struct {
    // 路由表
    routingTable      map[string]Handler
    // HTTP 协议请求转发handler
    httpForwarder     Handler
    // HTTP 请求处理handler
    httpHandler       http.Handler
}

// http 服务
type httpServer struct {
    // 默认兜底处理handler
    Server    stoppableServer
    // 接收TCP 到http 协议的网络转发请求
    Forwarder *httpForwarder
    // http 请求处理handler，在动态配置更新后，会更新处理handler
    Switcher  *middlewares.HTTPHandlerSwitcher
}
```

## 1. 创建入口实例

创建入口点，入口点可能有多个。

处理逻辑：

1. 开启网络监听
2. 创建TCP 请求路由
3. 创建http 协议请求处理handler
4. 创建tcp 协议请求处理器

```go
// 入口可能有多个，根据配置进行创建
func NewTCPEntryPoints(entryPointsConfig static.EntryPoints) (TCPEntryPoints, error) {
    serverEntryPointsTCP := make(TCPEntryPoints)
    for entryPointName, config := range entryPointsConfig {
        // ...
        serverEntryPointsTCP[entryPointName], err = NewTCPEntryPoint(ctx, config)
    }
    return serverEntryPointsTCP, nil
}

// 创建一个入口
func NewTCPEntryPoint(ctx context.Context, configuration *static.EntryPoint) (*TCPEntryPoint, error) {
    // ...
    // 网络监听
    listener, err := buildListener(ctx, configuration)
    
    // tcp 路由
    router := &tcp.Router{}

    // 创建http 服务
    httpServer, err := createHTTPServer(ctx, listener, configuration, true)
    
    // 设置路由中http 协议处理handler
    router.HTTPForwarder(httpServer.Forwarder)

    // tcp 请求处理器
    tcpSwitcher := &tcp.HandlerSwitcher{}
    tcpSwitcher.Switch(router)

    return &TCPEntryPoint{
        listener:               listener,
        switcher:               tcpSwitcher,
        transportConfiguration: configuration.Transport,
        httpServer:             httpServer,
        httpsServer:            httpsServer,
    }, nil
}
```

### 1.1 开启网络监听

开启入口端点网络监听。

```go
func buildListener(ctx context.Context, entryPoint *static.EntryPoint) (net.Listener, error) {
    listener, err := net.Listen("tcp", entryPoint.GetAddress())

    // ...
    return listener, nil
}
```

### 1.2 创建http 服务

创建http 服务。这里http 服务是默认处理handler，此时还没有读取动态配置，所有路由和服务配置还没有更新进来。

所以这里也只是初始化了默认处理handler。

```go
func createHTTPServer(ctx context.Context, ln net.Listener, configuration *static.EntryPoint, withH2c bool) (*httpServer, error) {
    // 404 处理handler
    httpSwitcher := middlewares.NewHandlerSwitcher(router.BuildDefaultHTTPRouter())

    // ip 白名单处理
    handler, err = forwardedheaders.NewXForwarded(
        configuration.ForwardedHeaders.Insecure,
        configuration.ForwardedHeaders.TrustedIPs,
        httpSwitcher)

    // 创建http 服务
    serverHTTP := &http.Server{
        Handler:      handler,
        ErrorLog:     httpServerLogger,
    }

    // tcp 连接到http 连接的转发处理
    listener := newHTTPForwarder(ln)
    // 开启http 服务
    go func() {
        err := serverHTTP.Serve(listener)
    }()
    
    return &httpServer{
        Server:    serverHTTP,
        Forwarder: listener,
        Switcher:  httpSwitcher,
    }, nil
}
```

封装TCP 协议网络连接到HTTP 协议网络连接的转发。

```go
func newHTTPForwarder(ln net.Listener) *httpForwarder {
    return &httpForwarder{
        Listener: ln,
        connChan: make(chan net.Conn),
    }
}

// ServeTCP 开启TCP 协议到HTTP 协议的转发
// 有网络请求时，将连接转发给http 协议处理
func (h *httpForwarder) ServeTCP(conn tcp.WriteCloser) {
    h.connChan <- conn
}

// Accept 获取HTTP 连接
func (h *httpForwarder) Accept() (net.Conn, error) {
    conn := <-h.connChan
    return conn, nil
}
```

### 1.3 设置HTTP 转发handler

HTTPForwarder 设置转发handler，用于转发http 连接。

```go
func (r *Router) HTTPForwarder(handler Handler) {
    r.httpForwarder = handler
}
```

### 1.4 更新TCP 协议转发handler

```go
// Switch 更新TCP 协议转发handler
func (s *HandlerSwitcher) Switch(handler Handler) {
    s.router.Set(handler)
}
```

实现ServeTCP 接口。

```go
// ServeTCP 处理TCP 请求
func (s *HandlerSwitcher) ServeTCP(conn WriteCloser) {
    handler := s.router.Get()
    h, ok := handler.(Handler)
    if ok {
        h.ServeTCP(conn)
    } else {
        conn.Close()
    }
}
```

## 2. 更新TCP 处理handler

在动态配置生效后，需要根据配置更新TCP 处理handler。主要配置更新是Router 和service 。

```go
// 更新TCP Router 配置
func (eps TCPEntryPoints) Switch(routersTCP map[string]*tcp.Router) {
    for entryPointName, rt := range routersTCP {
        eps[entryPointName].SwitchRouter(rt)
    }
}

// SwitchRouter 更新TCP 的router 处理handler
// 这里主要是根据配置更新路由表
// 路由表也即是请求处理handler
func (e *TCPEntryPoint) SwitchRouter(rt *tcp.Router) {
    // 取出已有http 协议转发器
    // 将已有转发器更新到路由器上，下面会更新EntryPoint 入口的路由器
    rt.HTTPForwarder(e.httpServer.Forwarder)

    // 从动态路由配置获取最新的HTTP 处理handler
    httpHandler := rt.GetHTTPHandler()
    if httpHandler == nil {
        // 如果没有就用默认handler
        httpHandler = router.BuildDefaultHTTPRouter()
    }

    // 将最新handler 更新到HTTP 服务上
    e.httpServer.Switcher.UpdateHandler(httpHandler)

    // 更新TCP 协议处理路由器
    e.switcher.Switch(rt)
}
```

```go
// UpdateHandler 更新http 转发器
func (h *HTTPHandlerSwitcher) UpdateHandler(newHandler http.Handler) {
    h.handler.Set(newHandler)
}
```

## 3. 开启入口服务 Start

入口服务有多个，这里循环开启。

每个入口内部处理流程：
1. 读取网络请求
2. 调用TCP 转发器 ServeTCP
3. 转发器取出TCP 路由器，调用路由器的 ServeTCP 

```go
func (eps TCPEntryPoints) Start() {
    for entryPointName, serverEntryPoint := range eps {
        // ...
        go serverEntryPoint.Start(ctx)
    }
}
```

开启一个入口服务。

```go
// Start 开启一个处理服务
func (e *TCPEntryPoint) Start(ctx context.Context) {
    // ...
    for {
        // 获取网络请求
        conn, err := e.listener.Accept()
        
        // ...
        writeCloser, err := writeCloser(conn)
        if err != nil {
            panic(err)
        }

        safe.Go(func() {
            // ...
            // 调用转发器处理请求
            e.switcher.ServeTCP(newTrackedConnection(writeCloser, e.tracker))
        })
    }
}

// ServeTCP 转码器将请求转给路由器处理
// 路由器根据路由配置转发请求到具体handler
func (s *HandlerSwitcher) ServeTCP(conn WriteCloser) {
    // router := &tcp.Router{}
    handler := s.router.Get()
    // 取出tpc.Router 进行TCP 请求处理
    h, ok := handler.(Handler)
    if ok {
        h.ServeTCP(conn)
    } else {
        conn.Close()
    }
}
```

## 参考资料

- github.com/containous/traefik/pkg/config/static/entrypoints.go

