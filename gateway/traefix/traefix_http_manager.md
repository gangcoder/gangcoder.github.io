<!-- ---
title: Traefix Http Service
date: 2020-07-21 23:57:32
category: showcode, gateway, traefix
--- -->

# Traefix Http Service

Http Service 管理，用于创建后端service 处理handler。这里分为http service 管理器和tcp service 管理器。

![](images/traefix_http_manager.svg)

主要代码：

```go
// 服务service 管理器创建HTTP 处理handler
sHandler, err := m.serviceManager.BuildHTTP(ctx, router.Service, rm)

// service 节点健康检查
serviceManager.LaunchHealthCheck()
```

主要数据结构：

```go
// 服务管理器接口
type serviceManager interface {
    BuildHTTP(rootCtx context.Context, serviceName string, responseModifier func(*http.Response) error) (http.Handler, error)
    LaunchHealthCheck()
}

// http 服务管理器
type Manager struct {
    // service 健康检查端点
    balancers map[string]healthcheck.Balancers
    // service 配置
    configs   map[string]*runtime.ServiceInfo
}

// 负载均衡状态管理
type LbStatusUpdater struct {
    BalancerHandler
    serviceInfo *runtime.ServiceInfo // can be nil
}

// BackendConfig 健康检查配置
type BackendConfig struct {
    Options
    name         string
    disabledURLs []backendURL
}
```

## 1. 创建HTTP 处理handler

service 处理实现。

BuildHTTP 根据service 配置创建http 服务的反向代理处理。

```go
// BuildHTTP 根据service 配置，创建http 处理handler
// 这里处理handler 是http 反向代理实现
func (m *Manager) BuildHTTP(rootCtx context.Context, serviceName string, responseModifier func(*http.Response) error) (http.Handler, error) {
    // ...

    var lb http.Handler
    // 基于负载均衡配置，创建service 处理handler
    switch {
    case conf.LoadBalancer != nil:
        lb, err = m.getLoadBalancerServiceHandler(ctx, serviceName, conf.LoadBalancer, responseModifier)
        // ...
    }

    return lb, nil
}
```

## 2. 轮询负载均衡

轮询负载均衡实现，这里是调用第三方库进行轮询处理。

增加处理逻辑：

1. 创建反向代理实例
2. 增加健康检查的负载均衡处理
3. 没有服务节点时兜底处理

```go
// 轮询负载均衡实现
func (m *Manager) getLoadBalancerServiceHandler(
    ctx context.Context,
    serviceName string,
    service *dynamic.ServersLoadBalancer,
    responseModifier func(*http.Response) error,
) (http.Handler, error) {
    // 创建反向代理实例
    fwd, err := buildProxy(service.PassHostHeader, service.ResponseForwarding, m.defaultRoundTripper, m.bufferPool, responseModifier)
    
    // 添加中间件
    handler, err := chain.Append(alHandler).Then(pipelining.New(ctx, fwd, "pipelining"))
    
    // 节点负载均衡处理
    balancer, err := m.getLoadBalancer(ctx, serviceName, service, handler)
    
    // 服务节点信息
    m.balancers[serviceName] = append(m.balancers[serviceName], balancer)

    // 没有服务节点时兜底处理，这也是一个中间件
    return emptybackendhandler.New(balancer), nil
}
```

### 2.1 HTTP 反向代理实现

基于golang 标准库实现http 反向代理。

```go
// 使用标准库的反向代理实现 ReverseProxy
func buildProxy(passHostHeader *bool, responseForwarding *dynamic.ResponseForwarding, defaultRoundTripper http.RoundTripper, bufferPool httputil.BufferPool, responseModifier func(*http.Response) error) (http.Handler, error) {
    // ...
    proxy := &httputil.ReverseProxy{
        Director: func(outReq *http.Request) {
            u := outReq.URL
            // ...
        },
        Transport:      defaultRoundTripper,
        // ...
    }

    return proxy, nil
}
```

### 2.2 节点轮询实现

节点轮询处理，轮询使用第三方库实现。

这里注意，在轮询实现的基础上，增加了健康检查结果处理，会根据健康检查的结果增删节点。

```go
func (m *Manager) getLoadBalancer(ctx context.Context, serviceName string, service *dynamic.ServersLoadBalancer, fwd http.Handler) (healthcheck.BalancerHandler, error) {
    // ...
    var options []roundrobin.LBOption

    // 轮询器
    lb, err := roundrobin.New(fwd, options...)
    
    // 增加健康检查处理
    lbsu := healthcheck.NewLBStatusUpdater(lb, m.configs[serviceName])
    // 更新节点
    err := m.upsertServers(ctx, lbsu, service.Servers)

    return lbsu, nil
}

// 负载均衡状态管理
// 根据节点健康检查结果，更新节点状态信息
func NewLBStatusUpdater(bh BalancerHandler, info *runtime.ServiceInfo) *LbStatusUpdater {
    return &LbStatusUpdater{
        BalancerHandler: bh,
        serviceInfo:     info,
    }
}

// 第一次创建实例后更新负载均衡节点
func (m *Manager) upsertServers(ctx context.Context, lb healthcheck.BalancerHandler, servers []dynamic.Server) error {
    // 更新后端节点
    for name, srv := range servers {
        // ...
        // 更新负载均衡节点
        lb.UpsertServer(u, roundrobin.Weight(1))
    }
    return nil
}
```

### 2.3 无节点兜底处理

没有服务节点时兜底处理，返回503 异常：

```go
func New(lb healthcheck.BalancerHandler) http.Handler {
    return &emptyBackend{next: lb}
}

// ServeHTTP 当没有service 节点时，返回503 信息
func (e *emptyBackend) ServeHTTP(rw http.ResponseWriter, req *http.Request) {
    // 没有可用负载均衡节点时，返回503
    if len(e.next.Servers()) == 0 {
        rw.WriteHeader(http.StatusServiceUnavailable)
        _, err := rw.Write([]byte(http.StatusText(http.StatusServiceUnavailable)))
        // ...
    } else {
        e.next.ServeHTTP(rw, req)
    }
}
```

## 3. 节点健康检查

LaunchHealthCheck 接口，实现检查service 各个负载均衡端点的健康状况。

1. 创建健康检查实例
2. 执行健康检查
3. 定时检查
4. 发起健康检查请求

```go
// LaunchHealthCheck 开启健康检查
func (m *Manager) LaunchHealthCheck() {
    backendConfigs := make(map[string]*healthcheck.BackendConfig)

    for serviceName, balancers := range m.balancers {
        // ...
        // 取到负载均衡节点配置
        service := m.configs[serviceName].LoadBalancer

        // 健康检查实例
        var backendHealthCheck *healthcheck.BackendConfig

        // 获取健康检查配置参数
        if hcOpts := buildHealthCheckOptions(ctx, balancers, serviceName, service.HealthCheck); hcOpts != nil {
            // ...
            hcOpts.Transport = m.defaultRoundTripper
            // 创建健康检查配置
            backendHealthCheck = healthcheck.NewBackendConfig(*hcOpts, serviceName)
        }

        if backendHealthCheck != nil {
            backendConfigs[serviceName] = backendHealthCheck
        }
    }

    // 执行健康检查任务
    healthcheck.GetHealthCheck().SetBackendsConfiguration(context.Background(), backendConfigs)
}

// github.com/containous/traefik/pkg/healthcheck/healthcheck.go
// 创建健康检查配置
func NewBackendConfig(options Options, backendName string) *BackendConfig {
    return &BackendConfig{
        Options: options,
        name:    backendName,
    }
}
```

### 3.1 开启健康检查配置

基于健康检查配置，开启运行健康检查：

```go
// github.com/containous/traefik/pkg/healthcheck/healthcheck.go
func (hc *HealthCheck) SetBackendsConfiguration(parentCtx context.Context, backends map[string]*BackendConfig) {
    hc.Backends = backends
    // ...
    for _, backend := range backends {
        currentBackend := backend
        // 执行健康检查
        safe.Go(func() {
            hc.execute(ctx, currentBackend)
        })
    }
}
```

### 3.2 定时检查健康状态

```go
// 根据配置，定时检查节点状态
func (hc *HealthCheck) execute(ctx context.Context, backend *BackendConfig) {
    // ...
    // 检查状态
    hc.checkBackend(ctx, backend)

    ticker := time.NewTicker(backend.Interval)
    defer ticker.Stop()
    for {
        select {
        case <-ticker.C:
            // 定时周期性检查状态
            hc.checkBackend(ctx, backend)
        }
    }
}

// 健康检查分别检查失效节点和健康节点
// 如果失效节点恢复，会转移为健康节点；同理健康节点检查失败会转为不可用节点
func (hc *HealthCheck) checkBackend(ctx context.Context, backend *BackendConfig) {
    // 获取所有正常节点
    enabledURLs := backend.LB.Servers()

    var newDisabledURLs []backendURL
    // 检查失效节点
    for _, disabledURL := range backend.disabledURLs {
        if err := checkHealth(disabledURL.url, backend); err == nil {
            // ...
            if err = backend.LB.UpsertServer(disabledURL.url, roundrobin.Weight(disabledURL.weight)); err 
        } else {
            // ...
            newDisabledURLs = append(newDisabledURLs, disabledURL)
        }
    }
    backend.disabledURLs = newDisabledURLs

    // 检查正常节点
    for _, enableURL := range enabledURLs {
        if err := checkHealth(enableURL, backend); err != nil {
            // ...
            // 检查节点失效，就从正常节点列表移除，并且记录为失效节点
            err := backend.LB.RemoveServer(enableURL)
            
            backend.disabledURLs = append(backend.disabledURLs, backendURL{enableURL, weight})
        }
    }
}
```

### 3.3 健康检查请求

发起健康检查http 请求。

```go
// checkHealth 发起请求
func checkHealth(serverURL *url.URL, backend *BackendConfig) error {
    // ...
    // 请求信息
    req, err := backend.newRequest(serverURL)
    req = backend.addHeadersAndHost(req)

    // 请求client
    client := http.Client{
        Timeout:   backend.Options.Timeout,
        Transport: backend.Options.Transport,
    }

    // 发起请求
    resp, err := client.Do(req)
    defer resp.Body.Close()

    // 根据状态码判断节点状态
    if resp.StatusCode < http.StatusOK || resp.StatusCode >= http.StatusBadRequest {
        return fmt.Errorf("received error status code: %v", resp.StatusCode)
    }

    return nil
}
```

## 参考资料

- github.com/containous/traefik/pkg/server/service/service.go
- github.com/containous/traefik/pkg/healthcheck/healthcheck.go
