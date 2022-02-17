<!-- ---
title: Traefix Tcp 路由管理器
date: 2020-07-25 23:35:48
category: showcode, gateway, traefix
--- -->

# Traefix Tcp 路由管理器

创建TCP 协议路由器

![](images/traefix_routermanager_tcp.svg)

主要调用逻辑：

```go
// TCP 协议管理器
svcTCPManager := tcp.NewManager(rtConf)

// TCP 路由管理器
rtTCPManager := routertcp.NewManager(rtConf, svcTCPManager, handlersNonTLS, handlersTLS, f.tlsManager)

// 创建TCP 路由器
routersTCP := rtTCPManager.BuildHandlers(ctx, f.entryPointsTCP)
```

主要代码结构：

```go
// Manager TCP 协议 service 管理器
type Manager struct {
    // TCP service 配置信息
    configs map[string]*runtime.TCPServiceInfo
}

// Manager TCP 路由管理器
type Manager struct {
    serviceManager *tcpservice.Manager
    httpHandlers   map[string]http.Handler
    httpsHandlers  map[string]http.Handler
    tlsManager     *traefiktls.Manager
    conf           *runtime.Configuration
}

// Router TCP 协议请求处理路由
type Router struct {
    routingTable      map[string]Handler
    httpForwarder     Handler
    httpHandler       http.Handler
}

// WRRLoadBalancer TCP 轮询负载均衡器
type WRRLoadBalancer struct {
    servers       []server
    lock          sync.RWMutex
    currentWeight int
    index         int
}
```

## 1. 创建TCP service 管理器

创建TCP service 管理器，用于处理TCP 请求。

```go
// github.com/containous/traefik/pkg/server/service/tcp/service.go
// NewManager 创建新的管理器
func NewManager(conf *runtime.Configuration) *Manager {
    return &Manager{
        configs: conf.TCPServices,
    }
}
```

### 1.1 创建TCP service 处理handler

创建基于service 的tcp handler。

```go
// BuildTCP 创建service 处理handler
func (m *Manager) BuildTCP(rootCtx context.Context, serviceName string) (tcp.Handler, error) {
    // ...
    switch {
    case conf.LoadBalancer != nil:
        // 创建TCP 负载均衡器
        loadBalancer := tcp.NewWRRLoadBalancer()
        // 处理到后端节点的代理
        for name, server := range conf.LoadBalancer.Servers {
            // ...
            // 创建TCP 反向代理
            handler, err := tcp.NewProxy(server.Address, duration)

            // ...
            // 添加负载均衡器节点
            loadBalancer.AddServer(handler)
        }
        return loadBalancer, nil
        // ...
    }
}
```

### 1.2 创建TCP 负载均衡器

创建负载均衡器。

```go
loadBalancer := tcp.NewWRRLoadBalancer()
// NewWRRLoadBalancer 创建轮询负载均衡器
func NewWRRLoadBalancer() *WRRLoadBalancer {
    return &WRRLoadBalancer{
        index: -1,
    }
}
```

### 1.3 创建TCP 反向代理

创建具体反向代理实现。

```go
handler, err := tcp.NewProxy(server.Address, duration)

// NewProxy 创建反向代理实现
func NewProxy(address string, terminationDelay time.Duration) (*Proxy, error) {
    tcpAddr, err := net.ResolveTCPAddr("tcp", address)
    
    // ...
    return &Proxy{target: tcpAddr, terminationDelay: terminationDelay}, nil
}

// ServeTCP 反向代理，将网络连接请求转发给后端服务
func (p *Proxy) ServeTCP(conn WriteCloser) {
    // 开启网络连接
    // ...
    connBackend, err := net.DialTCP("tcp", nil, p.target)
    
    // TCP 网络连接双向同步读写
    go p.connCopy(conn, connBackend, errChan)
    go p.connCopy(connBackend, conn, errChan)

    // ...
}
```

### 1.4 添加负载均衡器节点

负载均衡器添加一个反向代理器。

```go
loadBalancer.AddServer(handler)

// AddServer 添加负载均衡节点
func (b *WRRLoadBalancer) AddServer(serverHandler Handler) {
    w := 1
    b.AddWeightServer(serverHandler, &w)
}

// AddWeightServer
func (b *WRRLoadBalancer) AddWeightServer(serverHandler Handler, weight *int) {
    w := 1
    if weight != nil {
        w = *weight
    }
    b.servers = append(b.servers, server{Handler: serverHandler, weight: w})
}
```

## 2. 创建TCP 路由管理器

```go
rtTCPManager := routertcp.NewManager(rtConf, svcTCPManager, handlersNonTLS, handlersTLS, f.tlsManager)

// github.com/containous/traefik/pkg/server/router/tcp/router.go
// NewManager 创建TCP 路由管理器
func NewManager(conf *runtime.Configuration,
    serviceManager *tcpservice.Manager,
    httpHandlers map[string]http.Handler,
    httpsHandlers map[string]http.Handler,
    tlsManager *traefiktls.Manager,
) *Manager {
    return &Manager{
        serviceManager: serviceManager,
        httpHandlers:   httpHandlers,
        httpsHandlers:  httpsHandlers,
        tlsManager:     tlsManager,
        conf:           conf,
    }
}
```

## 3. 创建TCP 协议处理handler

调用路由管理器，生成TCP 协议处理handler。

```go
// BuildHandlers 创建入口的处理handler
func (m *Manager) BuildHandlers(rootCtx context.Context, entryPoints []string) map[string]*tcp.Router {
    entryPointsRouters := m.getTCPRouters(rootCtx, entryPoints)
    // 获取https 路由
    entryPointsRoutersHTTP := m.getHTTPRouters(rootCtx, entryPoints, true)

    entryPointHandlers := make(map[string]*tcp.Router)
    for _, entryPointName := range entryPoints {
        entryPointName := entryPointName
        // 获取TCP 路由配置
        routers := entryPointsRouters[entryPointName]
        // 创建TCP 处理handler
        handler, err := m.buildEntryPointHandler(ctx, routers, entryPointsRoutersHTTP[entryPointName], m.httpHandlers[entryPointName], m.httpsHandlers[entryPointName])
        // ..
        entryPointHandlers[entryPointName] = handler
    }
    return entryPointHandlers
}
```

创建入口的处理handler。

```go
func (m *Manager) buildEntryPointHandler(ctx context.Context, configs map[string]*runtime.TCPRouterInfo, configsHTTP map[string]*runtime.RouterInfo, handlerHTTP http.Handler, handlerHTTPS http.Handler) (*tcp.Router, error) {
    router := &tcp.Router{}
    // 设置http 处理handler
    router.HTTPHandler(handlerHTTP)

    // ...
    for routerName, routerConfig := range configs {
        // ...
        // 创建TCP 处理handler
        handler, err := m.serviceManager.BuildTCP(ctxRouter, routerConfig.Service)

        // 获取https host
        domains, err := rules.ParseHostSNI(routerConfig.Rule)
        // 处理https 协议handler
        for _, domain := range domains {
            switch {
            case routerConfig.TLS != nil:
                if routerConfig.TLS.Passthrough {
                    router.AddRoute(domain, handler)
                }
                // ...
            case domain == "*":
                router.AddCatchAllNoTLS(handler)
            }
        }
    }

    return router, nil
}
```

## 参考资料

- github.com/containous/traefik/pkg/tcp/router.go
- github.com/containous/traefik/pkg/server/service/tcp/service.go
