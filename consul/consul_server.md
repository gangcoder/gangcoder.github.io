<!-- ---
title: 服务端RPC 处理实现
date: 2019-07-26 01:06:36
category: src, consul
--- -->

# 服务端RPC 处理实现

consul 服务模式暴露的端点在引入包时通过 `init` 初始化函数注册。

server 相关逻辑包括：

1. 注册rpc 服务端端点，rpc 基于golang 提供的标准库rpc，使用msgpack 编解码
2. 注册函数将处理实例写入 `endpoints` slice

注册实现的主要服务包括：

1. Catalog
2. DiscoveryChain
3. Health
4. KVS

主要数据结构：

```go
// endpointFactory 返回一个RPC 端点
type factory func(s *Server) interface{}
```

## 1. 注册实现

```go
// github.com/hashicorp/consul/agent/consul/server_oss.go
func init() {
    registerEndpoint(func(s *Server) interface{} { return &Catalog{s, s.loggers.Named(logging.Catalog)} })
    registerEndpoint(func(s *Server) interface{} { return &DiscoveryChain{s} })
    registerEndpoint(func(s *Server) interface{} { return &Health{s} })
    registerEndpoint(func(s *Server) interface{} { return &KVS{s, s.loggers.Named(logging.KV)} })
}

func registerEndpoint(fn factory) {
    endpoints = append(endpoints, fn)
}
```

注册函数`registerEndpoint` 实现逻辑

```go
// endpoints 所有注册的端点
var endpoints []factory

// registerEndpoint 将所有的RPC 处理结构体都注册到本地变量 endpoints 中
func registerEndpoint(fn factory) {
    endpoints = append(endpoints, fn)
}
```

rpc 处理器收集完成后，在注册rpc 服务时将处理器注册到服务商。

## 参考资料

- github.com/hashicorp/consul/agent/consul/server.go

