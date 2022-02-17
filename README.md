# 优秀开源软件源代码阅读

## Golang 项目

| 项目名                                  | 用途            | 代码量 |
| -------------------------------------- | -------------- | ------ |
| [prometheus](prometheus/prometheus.md) | 监控和报警       | 46382  |
| [jaeger](jaeger/jaeger.md)             | 分布式链路跟踪    | 43809  |
| [etcd](etcd/etcd.md)                   | 服务发现         | 166360 |
| [NSQ](nsq/nsq.md)                      | 分布式消息队列系统 | - |
| [Echo](echo/echo.md)                   | web 应用框架     | - |

## grpc

* [grpc 远程调用协议](grpc/grpc.md)
  * [grpc 服务端服务实现](grpc/grpc_server.md)
  * [grpc 客户端调用实现](grpc/grpc_client.md)

## 微服务框架 go-micro

* [微服务框架](micro/micro.md)
  * [微服务框架](micro/go-micro/go-micro.md)
    * [创建微服务实现](micro/go-micro/go-micro_start.md)
    * [grpc 服务实现](micro/go-micro/go-micro_services_grpc.md)
    * [服务注册与发现实现](micro/go-micro/go-micro_registry.md)
    * [客户端grpc 实现](micro/go-micro/go-micro_client_grpc.md)
    * [客户端服务发现与节点选择实现](micro/go-micro/go-micro_client_registry.md)
    * [Go-Micro 客户端连接池实现](micro/go-micro/go-micro_client_pool.md)
    * [事件订阅与消费实现](micro/go-micro/go-micro_broker.md)

### 服务发现 Consul

* [Consul 服务发现](consul/consul.md)
  * [Consul 使用指南](consul/doc/consul_guide.md)
  * [Consul 安装与启动](consul/doc/consul_startup.md)
  * [Consul 组件结构](consul/consul_structure.md)
  * [Consul 代码结构](consul/consul_code.md)
  * [Consul 入口函数](consul/consul_main.md)
  * [Consul version 命令](consul/consul_command_version.md)
  * [Consul KV 命令](consul/consul_command_kv.md)
  * [Consul API 客户端](consul/consul_api.md)
  * [Consul agent 启动](consul/consul_agent.md)
  * [Consul agent client 模式](consul/consul_agent_client.md)
  * [Consul agent server 模式](consul/consul_agent_server.md)
  * [Consul 服务端RPC](consul/consul_server.md)
  * [Consul 服务端KV 实现](consul/consul_server_kv.md)
  * [Consul Health 实现](consul/consul_server_health.md)
  * [Consul Catelog 实现](consul/consul_server_catelog.md)
  * [Consul Raft 实现](consul/consul_raft.md)
  * [Consul Http 服务](consul/consul_http.md)
  * [Consul HTTP KV 接口实现](consul/consul_http_kv.md)


### 网关 Traefix

* [网关](gateway/gateway.md)
  * [网关介绍](gateway/gateway_introduce.md)
  * [Traefix 介绍](gateway/traefix/traefix_introduce.md)
  * [Traefix 安装与启动](gateway/traefix/traefix_startup.md)
  * [Traefix 配置](gateway/traefix/traefix_config.md)
  * [Traefix 服务入口](gateway/traefix/traefix_main.md)
  * [Traefix Provider 实现](gateway/traefix/traefix_provider.md)
  * [Traefix 配置监听](gateway/traefix/traefix_watcher.md)
  * [Traefix 管理创建器实现](gateway/traefix/traefix_internal_manager.md)
  * [Traefix 负载均衡实现](gateway/traefix/traefix_http_manager.md)
  * [Traefix EntryPoints](gateway/traefix/traefix_entrypoints.md)
  * [Traefix TCP router](gateway/traefix/traefix_tcp_router.md)
  * [Traefix 路由创建器实现](gateway/traefix/traefix_routerfactory.md)
  * [Traefix RouterManager](gateway/traefix/traefix_routermanager.md)
  * [Traefix TCP RouterManager](gateway/traefix/traefix_routermanager_tcp.md)
  * [Traefix Http 规则路由器](gateway/traefix/traefix_rulerouter.md)
  * [Traefix Balancer](gateway/traefix/traefix_balancer.md)

### 监控与告警 Prometheus

* [prometheus 监控和报警](prometheus/prometheus.md)
  * [Prometheus 指南](prometheus/prometheus_guide.md)
  * [Prometheus 安装与启动](prometheus/prometheus_startup.md)
  * [Prometheus 代码与目录结构](prometheus/prometheus_code.md)
  * [architecture](prometheus/prometheus/prometheus_architecture.md)
  * [Prometheus 结构图](prometheus/prometheus_structure.md)
  * [notifier](prometheus/server/prometheus_notifier.md)
  * [客户端SDK Golang](prometheus/client/client_golang.md)
    * [指标收集逻辑](prometheus/client/http_handler.md)
    * [注册与收集器](prometheus/client/registry.md)
    * [Metric 结构及实现](prometheus/client/metric.md)
    * [Desc 描述](prometheus/client/desc.md)
    * [Metric 注册与使用](prometheus/client/metric_register.md)
    * [Vec 指标](prometheus/client/vec.md)
    * [Gauge 指标](prometheus/client/metric_gauge.md)
    * [Summary 指标](prometheus/client/summary.md)
    * [Histogram 指标](prometheus/client/metric_histogram.md)
    * [Prometheus 服务API 封装](prometheus/client/api.md)
    * [Push 实现封装](prometheus/client/push.md)
  * [Prometheus 服务](prometheus/prometheus/prometheus.md)
    * [Prometheus 服务启动函数](prometheus/prometheus/prometheus_main.md)
    * [服务发现实现](prometheus/prometheus/prometheus_discovery.md)
    * [Scrape 抓取服务实现](prometheus/prometheus/prometheus_scrape.md)
    * [Notifier 告警通知实现](prometheus/prometheus/prometheus_notifier.md)
    * [Promql 查询引擎实现](prometheus/prometheus/prometheus_promql.md)
    * [Rules 规则服务实现](prometheus/prometheus/prometheus_rules.md)
    * [数据存储实现](prometheus/prometheus/prometheus_storage.md)
    * [Web 服务实现](prometheus/prometheus/prometheus_web.md)
  * [告警中心](prometheus/alertmanager/alertmanager.md)
    * [告警中心启动实现](prometheus/alertmanager/alertmanager_main.md)
    * [告警集群实现](prometheus/alertmanager/alertmanager_cluster.md)
    * [告警静默器实现](prometheus/alertmanager/alertmanager_silence.md)
    * [告警抑制实现](prometheus/alertmanager/alertmanager_inhibitor.md)
    * [标记器实现](prometheus/alertmanager/alertmanager_marker.md)
    * [告警内存存储实现](prometheus/alertmanager/alertmanager_memalerts.md)
    * [告警分发器实现](prometheus/alertmanager/alertmanager_dispatch.md)
    * [告警协调者实现](prometheus/alertmanager/alertmanager_coordinator.md)
    * [告警Web 实现](prometheus/alertmanager/alertmanager_web.md)


### 消息队列 NSQ

> 分布式消息队列系统

* [nsq 消息队列](nsq/nsq.md)
  * [NSQ 指南](nsq/nsq_guide.md)
  * [NSQ 安装与启动](nsq/nsq_startup.md)
  * [NSQ 代码与目录结构](nsq/nsq_code.md)
  * [NSQ 结构图](nsq/nsq_structure.md)
  * [NSQ 生产者实现](nsq/go-nsq/nsq_producer.md)
  * [NSQ 消费者实现](nsq/go-nsq/nsq_consumer.md)
  * [Lookupd 实现](nsq/nsq/nsq_lookupd.md)
  * [NSQD 服务实现](nsq/nsq/nsqd.md)

### 链路跟踪 Jaeger

> 分布式链路跟踪

* [jaeger](jaeger/jaeger.md)
  * [jaeger 指南](jaeger/jaeger_guide.md)
  * [opentracing 简介](jaeger/opentracing.md)
  * [opentracing 术语与数据模型](jaeger/opentracing_specification.md)
  * [Jaeger 安装与启动](jaeger/jaeger_startup.md)
  * [Jaeger 代码目录结构](jaeger/jaeger_code.md)
  * [Jaeger 上报示例](jaeger/jaeger_hotrod.md)
  * [Jaeger 结构图](jaeger/jaeger_structure.md)
  * [Jaeger Client Go 实现](jaeger/jaeger_client_go.md)
  * [Jaeger Client Reporter 实现](jaeger/jaeger_client_reporter.md)
  * [Jaeger Agent 实现](jaeger/jaeger_agent.md)
  * [Jaeger Agent Reporter 实现](jaeger/jaeger_agent_reporter.md)
  * [Jaeger Collector 实现](jaeger/jaeger_collector.md)
  * [Jaeger Collector 存储实现](jaeger/jaeger_collector_storage.md)
  * [Jaeger Collector 采样策略实现](jaeger/jaeger_collector_sample.md)
  * [Jaeger Query 实现](jaeger/jaeger_query.md)
  * [Jaeger Ingester 实现](jaeger/jaeger_ingester.md)

### Web 框架 Echo

> web 应用框架

* [echo web 框架](echo/echo.md)
  * [Echo 使用指南](echo/echo_guide.md)
  * [Echo 代码文件结构](echo/echo_directory.md)
  * [Echo 框架主逻辑](echo/echo_start.md)
  * [Echo 网络监听服务](echo/echo_netlisten.md)
  * [Echo 路由逻辑](echo/echo_router.md)
  * [Echo 中间件](echo/echo_middleware.md)
  * [Echo 请求处理](echo/echo_context.md)

## PHP 项目

* [Laravel 框架](laravel/laravel.md)
  * [HTTP 请求处理](laravel/laravel_http.md)
  * [Request 类实现](laravel/laravel_request.md)
  * [中间件实现](laravel/laravel_middleware.md)
  * [路由实现](laravel/laravel_route.md)
  * [Response 类实现](laravel/laravel_response.md)
  * [异常处理](laravel/laravel_exception.md)
  * [类和文件自动加载](laravel/laravel_load.md)

