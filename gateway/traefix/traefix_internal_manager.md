<!-- ---
title: Traefix 管理创建器实现
date: 2020-07-16 00:19:31
category: showcode, gateway, traefix
--- -->

# Traefix 管理创建器实现

实现Traefix 管理端API 和dashboard 等功能。

![](images/traefix_internal_manager.svg)

主要代码块:

```go
// 创建管理创建器
managerFactory := service.NewManagerFactory(*staticConfiguration, routinesPool, metricsRegistry)

// 创建service 管理器
serviceManager := f.managerFactory.Build(rtConf)

// 服务service 管理器创建服务HTTP 处理handler
sHandler, err := m.serviceManager.BuildHTTP(ctx, router.Service, rm)

// 管理器健康检查
serviceManager.LaunchHealthCheck()
```

主要数据结构：

```go
// ManagerFactory 管理端点创建器
type ManagerFactory struct {
    // API 端点
    api              func(configuration *runtime.Configuration) http.Handler
    // rest 配置提供
    restHandler      http.Handler
    // dashboard
    dashboardHandler http.Handler
    // 指标端点
    metricsHandler   http.Handler
    // ping 检查处理
    pingHandler      http.Handler
}

// http 服务管理器
type Manager struct {
    // 服务均衡节点配置
    balancers map[string]healthcheck.Balancers
    // 服务配置
    configs   map[string]*runtime.ServiceInfo
}

// 包含内部处理服务的service 管理器
type InternalHandlers struct {
    api        http.Handler
    dashboard  http.Handler
    rest       http.Handler
    prometheus http.Handler
    ping       http.Handler
    // http service 管理器
    serviceManager
}

// service 管理器接口
type serviceManager interface {
    BuildHTTP(rootCtx context.Context, serviceName string, responseModifier func(*http.Response) error) (http.Handler, error)
    LaunchHealthCheck()
}
```

## 1. 创建管理创建器

创建管理器创建器实例。

1. traefix 管理API 端点
2. rest 配置提供处理处理
3. dashboard 处理
4. 指标端点处理
5. ping 检查处理

```go
// 创建管理创建器
// 这里会实现traefix 内部服务处理
func NewManagerFactory(staticConfiguration static.Configuration, routinesPool *safe.Pool, metricsRegistry metrics.Registry) *ManagerFactory {
    factory := &ManagerFactory{
        metricsRegistry:     metricsRegistry,
        defaultRoundTripper: setupDefaultRoundTripper(staticConfiguration.ServersTransport),
        routinesPool:        routinesPool,
    }
    // ...
    factory.api = api.NewBuilder(staticConfiguration)
    factory.pingHandler = staticConfiguration.Ping

    return factory
}
```

### 1.1 Traefix API 处理

暴露Traefix 管理端点。

```go
// NewBuilder
func NewBuilder(staticConfig static.Configuration) func(*runtime.Configuration) http.Handler {
    return func(configuration *runtime.Configuration) http.Handler {
        return New(staticConfig, configuration).createRouter()
    }
}
```

注册API 端点接口。

```go
// createRouter 注册API 端点
func (h Handler) createRouter() *mux.Router {
    router := mux.NewRouter()
    // ...

    router.Methods(http.MethodGet).Path("/api/rawdata").HandlerFunc(h.getRuntimeConfiguration)
    router.Methods(http.MethodGet).Path("/api/overview").HandlerFunc(h.getOverview)
    router.Methods(http.MethodGet).Path("/api/entrypoints").HandlerFunc(h.getEntryPoints)
    router.Methods(http.MethodGet).Path("/api/http/routers").HandlerFunc(h.getRouters)
    router.Methods(http.MethodGet).Path("/api/http/services").HandlerFunc(h.getServices)

    return router
}
```

### 1.2 Traefix ping 接口处理

实现Traefix ping 接口处理逻辑。

```go
// staticConfiguration.Ping
func (h *Handler) ServeHTTP(response http.ResponseWriter, request *http.Request) {
    statusCode := http.StatusOK
    // ...
    response.WriteHeader(statusCode)
    fmt.Fprint(response, http.StatusText(statusCode))
}
```

## 2. 创建service 管理器

管理创建器创建服务管理器。

这里会创建http service 管理器。同时为了处理traefix 内部逻辑的请求，会将http service 包一层，因此会创建一个Internal 管理器。Internal 管理器如果发现请求是traefix 内部请求，会使用内部处理handler 进行处理。

```go
// Build 创建服务管理器
func (f *ManagerFactory) Build(configuration *runtime.Configuration) *InternalHandlers {
    // 创建http service 管理器
    // pkg/server/service/service.go
    svcManager := NewManager(configuration.Services, f.defaultRoundTripper, f.metricsRegistry, f.routinesPool)
    // 创建包含内部处理handler 的service 管理器；具体service 相关功能会交给http service 管理器实现
    return NewInternalHandlers(f.api, configuration, f.restHandler, f.metricsHandler, f.pingHandler, f.dashboardHandler, svcManager)
}

// 创建http 服务管理器
func NewManager(configs map[string]*runtime.ServiceInfo, defaultRoundTripper http.RoundTripper, metricsRegistry metrics.Registry, routinePool *safe.Pool) *Manager {
    return &Manager{
        routinePool:         routinePool,
        metricsRegistry:     metricsRegistry,
        bufferPool:          newBufferPool(),
        defaultRoundTripper: defaultRoundTripper,
        balancers:           make(map[string]healthcheck.Balancers),
        configs:             configs,
    }
}
```

```go
// 创建包含内部处理handler 的service 管理器
func NewInternalHandlers(api func(configuration *runtime.Configuration) http.Handler, configuration *runtime.Configuration, rest http.Handler, metricsHandler http.Handler, pingHandler http.Handler, dashboard http.Handler, next serviceManager) *InternalHandlers {
    // ...
    return &InternalHandlers{
        api:            apiHandler,
        dashboard:      dashboard,
        rest:           rest,
        prometheus:     metricsHandler,
        ping:           pingHandler,
        serviceManager: next,
    }
}
```

## 3. 创建处理handler

创建http 请求处理handler。

```go
// BuildHTTP 包含内部处理handler 的服务管理器
// 如果判断 service 名称是内部服务，则直接使用内部服务进行处理
func (m *InternalHandlers) BuildHTTP(rootCtx context.Context, serviceName string, responseModifier func(*http.Response) error) (http.Handler, error) {
    // 如果是内部服务，就是用内部处理handler 进行处理
    if strings.HasSuffix(serviceName, "@internal") {
        return m.get(serviceName)
    }

    // 否则调用http service 进行处理
    // 具体在http manager 中解析
    return m.serviceManager.BuildHTTP(rootCtx, serviceName, responseModifier)
}
```

### 3.1 Traefix 内部请求处理

如果是内部服务，就使用内部处理handler 进行处理。

```go
// strings.HasSuffix(serviceName, "@internal")
// m.get(serviceName)

// 内部处理handler
func (m *InternalHandlers) get(serviceName string) (http.Handler, error) {
    switch serviceName {
    case "api@internal":
        // 内部API 接口
        return m.api, nil
    case "dashboard@internal":
        // 仪表盘
        return m.dashboard, nil
    case "ping@internal":
        // ping 接口
        return m.ping, nil
        // ...
    }
}
```

## 参考资料

- github.com/containous/traefik/pkg/healthcheck/healthcheck.go