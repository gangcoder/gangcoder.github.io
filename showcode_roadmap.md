<!-- ---
title: showcode roadmap
date: 2019-12-11 21:29:27
category: showcode
--- -->

## 目标

项目目标：
1. 阅读优秀软件源代码，以完成项目代码笔记和逻辑图为阅读完成的评价标准。通过阅读源码的方式提升编程技术能力。
2. 其中Golang 语言围绕微服务相关软件阅读源码，并且以出书为目标，按照书籍方式组织代码笔记，后续可考虑开发配套教学视频，分享软件使用与源码实现。

实现方式：整理和积累优秀开源项目源码实现阅读笔记和逻辑图，不断拓宽对大型软件系统源码实现的见识，提升技术能力。

## 实现路线

实现分3 个关注阶段：

1. 阶段一：重点实现以Golang 为主的微服务架构相关优秀项目源码阅读
2. 阶段二：重点是Golang, Java, PHP 相关优秀框架等大型项目源码阅读
3. 阶段三：重点Golang, Java, PHP 常用类库工具和语言本身源码实现阅读

### 阶段一 微服务开源项目

高优先级

1. Prometheus 监控和报警
2. nsq 消息系统
3. Jaeger 链路跟踪
4. echo web 框架
5. consul 服务发现
6. 网关项目阅读 gateway
7. 微服务项目阅读 micro
8. cobra 终端库使用
9. cron 定时任务库/分布式定时任务
10. minio 存储系统
11. etcd 服务发现
12. grpc 通信协议

### 阶段二 框架与大型开源项目

1. Beego Golang Web 开发框架
2. Spring Java Web 开发框架
3. Laravel PHP Web 开发框架

### 阶段三 常用类库

Golang: 

1. golang 标准库
   1. net http
   2. context
2. redis
   1. redigo redis 客户端
3. router
   1. httprouter golang 路由库
4. zap 日志库
5. rocketmq client_golang
6. gorm DB库


## 2019 完成情况

### 2019 已完成工作

1. 整理showcode 项目
2. cobra 源码阅读
3. cron 源码阅读
4. redigo 源码阅读
5. prometheus
   1. client_golang 代码阅读与整理
   2. prometheus 服务程序代码阅读与整理
   3. prometheus alertmanager 代码阅读与整理
   4. prometheus pushgateway 代码阅读
   5. Prometheus 官方文档阅读
6.  consul 代码阅读
    1. consul 使用文档整理
    2. consul 启动代码与agent 代码阅读
    3. consul kv 代码阅读
    4. consul 启动代码整理
7.  etcd 代码阅读
    1. etcd 中文官方文档阅读
    2. etcd ctl 逻辑和文档整理
    3. etcd proxy 逻辑和文档
    4. etcd etcd 逻辑和文档整理
8.  jaeger 代码阅读
    1. jaeger 服务端代码阅读
    2. jaeger agent 代码阅读
    3. jaeger storage 代码阅读
    4. jaeger client-go 代码逻辑
    5. jaeger hotrod 代码逻辑阅
    6. jaeger report 代码阅读
    7. jaeger query 文档和结构图整理
    8. jaeger ingester 文档整理
    9. jaeger collecotr 文档和结构图整理
    10. jaeger 文档阅读

### 2020 待推进完善

Prometheus 项目待完成事项：

1. prometheus 使用方式笔记
2. prometheus 官方文档笔记
3. pushgateway 代码阅读与逻辑图整理
4. node_exporter 代码阅读与逻辑图整理

项目笔记与逻辑整理：

1. consul
2. etcd
3. jaeger

## 2020 计划

> 为达成总体阶段一 微服务项目阅读目标，2020 年应该完成的工作

### 2020 目标与时间安排

需要完成阅读和代码逻辑整理的项目：

1. Prometheus
2. NSQ
3. Jaeger
4. Echo
5. Consul
6. 网关项目阅读 gateway
7. 微服务项目阅读 micro
8. grpc
9. rocketmq client_golang
10. 阅读1~2 个优秀java 开源项目

2020Q1

1. Prometheus 1~2月
2. NSQ 开始 3月

2020Q2

1. NSQ 整理完成 4月
2. Jaeger 整理 4月
3. Echo 整理完成 5月
   1. micro 代码快速学习
4. Consul 整理 6月
   1. gateway 代码快速学习

2020Q3

1. 网关项目阅读 gateway 7月
   1. 网关概念
   2. traefix 源码阅读
2. 微服务项目阅读 micro 8月
   1. go-micro 源码阅读
   2. go-kit 使用整理
3. 微服务组件概念（架构深化学习） 9月
   1. 微服务与云原生架构学习路线梳理
   2. 微服务
   3. 云原生
   4. Service Mesh istio 文档初步了解
   5. k8s
   6. go-micro 源码阅读
   7. grpc 源码阅读

2020Q4

Golang 相关项目源码阅读，查漏补缺
1. golang 项目补缺 10月
   1. gorm
   2. redisgo https://github.com/go-redis/redis
   3. cron
   4. cobra
2. golang 项目补缺 11 月
   1. http
   2. httprouter
   3. zap
   4. minio
   5. rocketmq client_golang
   6. influxdata
3. laravel 12 月
4. java 开源项目阅读 12月
   1. spring boot
   2. 基于spring boot 实现的项目源码阅读 https://github.com/macrozheng/mall

### 2020 已完成工作

### 2021 待推荐完善
