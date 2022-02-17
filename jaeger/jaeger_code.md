<!-- ---
title: jaeger 分布式链路分析
date: 2018-10-16 10:38:35
category: architecture, jaeger
--- -->

jaeger 分布式链路分析

## 代码量

`github.com/jaegertracing/jaeger` 服务端代码量：

```
$ cloc ./ --exclude-dir="vendor" --include-ext="go"
-------------------------------------------------------------------------------
Language                     files          blank        comment           code
-------------------------------------------------------------------------------
Go                             598           9630          10701          64553
-------------------------------------------------------------------------------
SUM:                           598           9630          10701          64553
-------------------------------------------------------------------------------
```

`github.com/jaegertracing/jaeger-client-go` golang 客户端SDK 代码量:

```
$ cloc ./ --exclude-dir="vendor" --include-ext="go"
-------------------------------------------------------------------------------
Language                     files          blank        comment           code
-------------------------------------------------------------------------------
Go                             152           3187           3442          20928
-------------------------------------------------------------------------------
SUM:                           152           3187           3442          20928
-------------------------------------------------------------------------------
```

## 目录结构

```
github.com/jaegertracing
├── jaeger 服务端
│   ├── cmd 各组件入口
│   │   ├── agent 安装在应用实例节点上，收集实例上报的tracer 数据，然后上报到collector
│   │   │   ├── app
│   │   │   │   ├── agent.go Agent 结构体
│   │   │   │   ├── builder.go 创建结构体实例
│   │   │   │   ├── configmanager 采样策略查询实现
│   │   │   │   ├── httpserver 提供http 服务，获取采样策略
│   │   │   │   ├── processors 服务处理实现
│   │   │   │   ├── reporter 上报实现
│   │   │   │   └── servers agent 服务逻辑实现
│   │   │   └── main.go
│   │   ├── all-in-one 一键启动所有依赖组件
│   │   │   └── main.go
│   │   ├── collector 接收agent 和client 发送的span 数据，并且将这些数据写入存储中
│   │   │   ├── app
│   │   │   │   ├── builder 创建span 处理handler
│   │   │   │   ├── grpc_handler.go  基于grpc 协议的服务实现
│   │   │   │   ├── grpcserver 创建grpc 服务逻辑
│   │   │   │   ├── http_handler.go 基于http 协议的服务实现
│   │   │   │   ├── sampling 采样策略实现
│   │   │   │   ├── sanitizer 数据校验与处理
│   │   │   │   ├── span_processor.go span 数据处理，先将服务handle 收到的数据写入channel，再消费channel 将数据写入底层存储
│   │   │   │   ├── thrift_span_handler.go 基于thrift 协议的服务处理实现
│   │   │   │   └── zipkin 基于zipkin 协议的服务实现
│   │   │   └── main.go
│   │   ├── ingester 消费kafka 队列中的traces 数据，写入底层存储中
│   │   │   ├── app
│   │   │   │   ├── builder 创建消费实例
│   │   │   │   ├── consumer 消费kafka 逻辑实现
│   │   │   │   └── processor 获取到span 数据后的处理逻辑实现
│   │   │   └── main.go
│   │   ├── query 从底层存储中查询traces 和span 数据
│   │   │   ├── app
│   │   │   │   ├── grpc_handler.go 处理grpc 协议的查询请求
│   │   │   │   ├── http_handler.go 处理http 协议的查询请求
│   │   │   │   ├── querysvc 查询底层存储逻辑实现
│   │   │   │   └── server.go 查询服务的监听逻辑实现
│   │   │   └── main.go
│   ├── examples 示例代码
│   │   └── hotrod 示例程序，产生调用链数据，用于学习和测试jaeger
│   ├── model 数据结构体定义和数据辅助函数，以及接口生成代码定义
│   ├── pkg 依赖包调用逻辑
│   ├── plugin 插件化的底层存储依赖
│   │   ├── sampling 采样策略存储组件
│   │   │   ├── internal
│   │   │   └── strategystore
│   │   └── storage 依赖的底层存储调用实现逻辑
│   │       ├── factory.go 创建底层存储实例
│   │       ├── badger 
│   │       ├── cassandra
│   │       ├── es
│   │       ├── grpc
│   │       ├── kafka
│   │       └── memory
│   ├── ports 默认使用的端口配置
│   ├── proto-gen 生成的grpc 调用代码
│   ├── storage 底层存储实现的接口定义
└── jaeger-client-go
    ├── config 配置结构体实现，以及创建trace 的逻辑也在这里
    ├── crossdock 集成测试组件和代码
    ├── internal 采样策略和baggage 部分实现逻辑
    ├── log 日志记录库
    ├── rpcmetrics 调用指标数据统计逻辑
    ├── transport 数据发送到agent 或者collector 的实现逻辑
    ├── utils 工具库
    ├── constants.go 内部常量
    ├── context.go 上下文调用关系处理
    ├── metrics.go 使用的指标初始化逻辑
    ├── observer.go 观察逻辑实现
    ├── propagation.go 数据编码和传输处理
    ├── reporter.go 数据上报到agent 或collector 逻辑实现
    ├── sampler.go 数据是否需要采集逻辑实现
    ├── span.go 创建和操作span 逻辑实现
    ├── tracer.go 创建和操作trace 逻辑实现
    ├── transport.go 数据传输接口
    └── transport_udp.go 数据通过udp 传输
```

