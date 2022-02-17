<!-- ---
title: traefix 介绍
date: 2020-07-11 12:18:13
category: showcode, gateway, traefix
--- -->

# traefix 介绍

Traefik 是一个开源的可以使服务发布变得轻松有趣的边缘路由器。它负责接收你系统的请求，然后使用合适的组件来对这些请求进行处理。

除了众多的功能之外，Traefik 的与众不同之处还在于它会自动发现适合你服务的配置。当 Traefik 在检查你的服务时，会找到服务的相关信息并找到合适的服务来满足对应的请求。

使用 Traefik，不需要维护或者同步一个独立的配置文件：因为一切都会自动配置，实时操作的（无需重新启动，不会中断连接）。使用 Traefik，你可以花更多的时间在系统的开发和新功能上面，而不是在配置和维护工作状态上面花费大量时间。

Traefik 是一个边缘路由器，是你整个平台的大门，拦截并路由每个传入的请求：它知道所有的逻辑和规则，这些规则确定哪些服务处理哪些请求；传统的反向代理需要一个配置文件，其中包含路由到你服务的所有可能路由，而 Traefik 会实时检测服务并自动更新路由规则，可以自动服务发现。


## 路由与服务

首先，当启动 Traefik 时，需要定义 entrypoints（入口点），然后，根据连接到这些 entrypoints 的路由来分析传入的请求，来查看他们是否与一组规则相匹配，如果匹配，则路由可能会将请求通过一系列中间件转换过后再转发到你的服务上去。在了解 Traefik 之前有几个核心概念我们必须要了解：

1. `Providers` 用来自动发现平台上的服务，可以是编排工具、容器引擎或者 key-value 存储等，比如 Docker、Kubernetes、File
2. `Entrypoints` 监听传入的流量（端口等…），是网络入口点，它们定义了接收请求的端口（HTTP 或者 TCP）。
3. `Routers` 分析请求（host, path, headers, SSL, …），负责将传入请求连接到可以处理这些请求的服务上去。
4. `Services` 将请求转发给你的应用（load balancing, …），负责配置如何获取最终将处理传入请求的实际服务。
5. `Middlewares` 中间件，用来修改请求或者根据请求来做出一些判断（authentication, rate limiting, headers, …），中间件被附件到路由上，是一种在请求发送到你的服务之前（或者在服务的响应发送到客户端之前）调整请求的一种方法。


## EntryPoints

EntryPoints 是Traefik的网络入口点. 它们定义了将接收数据包的端口，以及侦听TCP还是UDP。

EntryPoints是静态配置的一部分. 可以通过使用文件（TOML或YAML）或CLI参数来定义它们.

```
## Static configuration
[entryPoints]
  [entryPoints.web]
    address = ":80"
```


## Routers

将请求连接到服务。

路由器负责将传入请求连接到可以处理这些请求的服务. 在此过程中，路由器可以使用中间件来更新请求，或者在将请求转发到服务之前采取行动。

```toml
## Dynamic configuration
[http.routers]
  [http.routers.my-router]
    rule = "Path(`/foo`)"
    service = "service-foo"
```

### Rule

规则是一组配置有值的匹配器，这些值确定特定请求是否匹配特定条件. 如果该规则得到验证，则路由器将变为活动状态，调用中间件，然后将请求转发到服务.

```toml
rule = "Host(`example.com`)"
```

1. `Host(`example.com`, ...)` 默认情况下，等效于HostHeader 和 HostSNI规则. 有关更多详细信息，请参见域前沿和迁移指南 .
2. `Method(`GET`, ...)` 检查请求方法是否为给定methods （ GET ， POST ， PUT ， DELETE ， PATCH ）
3. `Path(`/path`, `/articles/{cat:[a-z]+}/{id:[0-9]+}`, ...)` 匹配确切的请求路径. 它接受文字和正则表达式路径的序列.
4. `PathPrefix(`/products/`, `/articles/{cat:[a-z]+}/{id:[0-9]+}`)` 匹配请求前缀路径. 它接受一系列文字和正则表达式前缀路径.


### Middlewares

您可以将中间件列表附加到每个HTTP路由器. 仅当规则匹配时，并且在将请求转发到服务之前，中间件才会生效.

```toml
## Dynamic configuration
[http.routers]
  [http.routers.my-router]
    rule = "Path(`/foo`)"
    # declared elsewhere
    middlewares = ["authentication"]
    service = "service-foo"
```

### Service

每个请求最终都必须由服务处理，这就是为什么每个路由器定义都应包括服务目标的原因，该服务目标基本上是将请求传递到的目标.

## Services

Services 负责配置如何到达最终将处理传入请求的实际服务.

```toml
## Dynamic configuration
[http.services]
  [http.services.my-service.loadBalancer]

    [[http.services.my-service.loadBalancer.servers]]
      url = "http://<private-ip-server-1>:<private-port-server-1>/"
    [[http.services.my-service.loadBalancer.servers]]
      url = "http://<private-ip-server-2>:<private-port-server-2>/"
```

### Servers Load Balancer

负载均衡器能够在程序的多个实例之间负载均衡请求.

每个服务都有一个负载平衡器，即使只有一台服务器将流量转发到该服务器也是如此.


### Servers

服务器声明程序的单个实例. url选项指向特定实例.


### Health Check

配置运行状况检查，以从负载平衡循环中删除不正常的服务器. 只要Traefik服务器将2XX到3XX之间的状态代码返回到运行状况检查请求（在每个interval执行一次），它们就会认为您的服务器运行状况良好.

```toml
## Dynamic configuration
[http.services]
  [http.services.Service-1]
    [http.services.Service-1.loadBalancer.healthCheck]
      path = "/health"
      port = 8080
```

## 中间件

中间件连接到路由器，是一种在请求发送到您的服务之前（或在服务的答案发送到客户端之前）调整请求的方法.

Traefik中有几种可用的中间件，一些可以修改请求，标头，一些负责重定向，一些添加身份验证，等等.

中间件可以链式组合以适应各种情况.

```toml
# As TOML Configuration File
[http.routers]
  [http.routers.router1]
    service = "myService"
    middlewares = ["foo-add-prefix"]
    rule = "Host(`example.com`)"

[http.middlewares]
  [http.middlewares.foo-add-prefix.addPrefix]
    prefix = "/foo"

[http.services]
  [http.services.service1]
    [http.services.service1.loadBalancer]

      [[http.services.service1.loadBalancer.servers]]
        url = "http://127.0.0.1:80"
```


BasicAuth

```toml
# Declaring the user list
[http.middlewares]
  [http.middlewares.test-auth.basicAuth]
  users = [
    "test:$apr1$H6uskkkW$IgXLP6ewTrSuBkTrqE8wj/", 
    "test2:$apr1$d9hr9HBB$4HxwgUir3HP4EsggP/QNo0",
  ]
```


## 指标

指标体系

Traefik支持4种指标后端：

1. Datadog
1. InfluxDB
1. Prometheus
1. StatsD


要启用Prometheus：

```
[metrics]
  [metrics.prometheus]
```

## Tracing
 
可视化请求流

跟踪系统使开发人员可以可视化其基础架构中的呼叫流程.

Traefik使用OpenTracing，这是一种为分布式跟踪设计的开放标准.

Traefik支持六个跟踪后端：

1. Jaeger
1. Zipkin
1. Datadog
1. Instana
1. Haystack
1. Elastic

要启用Jaeger：

```
[tracing]
  [tracing.jaeger]
```


## 参考资料

- https://docs.traefik.io/
- https://www.qikqiak.com/post/traefik-2.1-101/
