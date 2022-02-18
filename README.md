# 个人源码阅读笔记

- 徐刚
- hubeixugang@163.com

## Golang

* [echo web 框架](echo/echo.md)
* [nsq 消息队列](nsq/nsq.md)
* [jaeger](jaeger/jaeger.md)
* [prometheus 监控和报警](prometheus/prometheus.md)
* [Consul 服务发现](consul/consul.md)
* [Traefix 网关](gateway/gateway.md)
* [微服务框架](micro/micro.md)
* [Grpc 远程调用协议](grpc/grpc.md)
* [Gorm 数据库](gorm/gorm.md)
* [go-redis Redis](redis/go-redis.md)
* [cobra 终端库使用](cobra/cobra.md)
* [cron 定时任务库](cron/robfigcron.md)
* [zap log 库](zap/zap.md)
* [MinIO 对象存储](minio/minio.md)
* [redigo 连接库](redigo/redigo.md)
* [etcd 服务发现](etcd/etcd.md)

## PHP 项目

* [Laravel 框架](laravel/laravel.md)

## Java 项目

* [Spring 框架](springcode/springcode.md)
* [SpringMVC 源码](springmvccode/springmvccode.md)
* [SpringBoot 框架](springbootcode/springbootcode.md)

## 部分逻辑调用图展示

NSQD 服务实现:

![](/nsq/nsq/images/nsqd.jpg)

Echo 请求处理:

![](/echo/images/echo_context.svg)

Spring XML 应用上下文启动实现:

![](/springcode/draw/spring_xml_application_context.svg)

Spring AOP 代理实例化与调用:

![](/springcode/draw/spring_aop_invoke.svg)

SpringBoot SpringApplication 初始化过程:

![](/springbootcode/draw/springbootcode_springapplication.svg)
