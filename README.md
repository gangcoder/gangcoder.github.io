# 个人源码阅读笔记

- 徐刚
- hubeixugang@163.com

始于2018年，持续源码阅读与调用图绘制记录。

## Golang

* [Echo Web 框架](/echo/echo.md)
* [NSQ 消息队列](/nsq/nsq.md)
* [Jaeger](/jaeger/jaeger.md)
* [Prometheus 监控和报警](prometheus/prometheus.md)
  * [客户端SDK Golang](prometheus/client/client_golang.md)
  * [Prometheus 服务](prometheus/prometheus/prometheus.md)
  * [告警中心](prometheus/alertmanager/alertmanager.md)
* [Consul 服务发现](/consul/consul.md)
* [网关](gateway/gateway.md)
  * [Traefix 网关](gateway/traefix/traefix.md)
* [微服务框架](/micro/micro.md)
  * [Go Micro 微服务框架](micro/go-micro/go-micro.md)
* [Grpc 远程调用协议](/grpc/grpc.md)
* [Gorm 数据库](/gorm/gorm.md)
* [go-redis Redis](/redis/go-redis.md)
* [cobra 终端库使用](/cobra/cobra.md)
* [cron 定时任务库](/cron/robfigcron.md)
* [zap log 库](/zap/zap.md)
* [etcd 服务发现](/etcd/etcd.md)

## PHP

* [Laravel 框架](/laravel/laravel.md)

## Java

* [Spring 框架](/springcode/springcode.md)
* [SpringMVC 源码](/springmvccode/springmvccode.md)
* [SpringBoot 框架](/springbootcode/springbootcode.md)


## 逻辑调用图演进展示

4 张图分别是初期绘图尝试，前期绘制方式确定，中期手工加插件绘图，以及当前基于手工整理后通过mxgraph 生成绘图。

![](/nsq/nsq/draw/nsq_structure.svg)

![](/echo/images/echo_context.svg)

Spring AOP 代理实例化与调用:

![](/springcode/draw/spring_aop_invoke.svg)

SpringBoot SpringApplication 初始化过程:

![](/springbootcode/draw/springbootcode_springapplication.svg)

