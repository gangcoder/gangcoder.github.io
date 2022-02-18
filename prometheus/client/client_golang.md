<!-- ---
title: client_golang
date: 2019-05-17 00:51:33
category: src, prometheus, client
--- -->

client_golang

## 内容

客户端数据是什么样的，怎样产生，怎么存储，怎么暴露给服务端拉取。

代码基于 github.com/prometheus/client_golang  v1.4.1 版本。

- 快速使用 Prometheus 指标
- [指标收集逻辑](/prometheus/client/http_handler.md) 详解应用中暴露 prometheus 抓取端点的处理过程，提供整体逻辑思路和处理流程
- [注册与收集器](/prometheus/client/registry.md) metric 注册和收集处理逻辑
- [Metric 结构及实](/prometheus/client/metric.md) 指标需要实现的功能，包括：指标数据功能，描述，收集功能，指标通用数据写入功能
- [Desc 描描述](/prometheus/client/desc.md) 描述指标属性
- [Metric 注册与使用](/prometheus/client/metric_register.md) Metric 指标的初始化，注册与数据收集细节详解
- [Vec 指标](/prometheus/client/vec.md) Vec 类指标的结构与处理逻辑
- [Gauge 指标](/prometheus/client/metric_gauge.md) gauge 指标的处理
- [Summary 指标](/prometheus/client/metric_summary.md) summary 指标的处理
- [Histogram 指标](/prometheus/client/metric_histogram.md) histogram 指标的处理
- [Prometheus 服务API 封装](/prometheus/client/api.md) api 提供请求Prometheus server 端接口的http 封装
- [Push 实现封装](/prometheus/client/push.md) push 提供推送 metrics 到 Pushgateway 的功能封装

项目目录结构：

```
github.com/prometheus/client_golang
├── api 封装了请求prometheus 服务程序的功能封装
├── examples client_golang 使用示例代码
└── prometheus 
    ├── promauto 创建Metric 并且注册到默认注册器
    ├── promhttp http 端口服务及http 请求指标记录的中间件实现
    ├── push 推送 metrics 到 Pushgateway 的功能封装
    ├── collector.go 
    ├── counter.go counter 指标实现
    ├── desc.go desc 实现
    ├── gauge.go gauge 指标实现
    ├── histogram.go histogram 指标实现
    ├── labels.go
    ├── metric.go
    ├── observer.go
    ├── registry.go 注册器和收集器实现
    ├── summary.go summary 指标实现
    ├── value.go
    └── vec.go vec 指标实现
```

## 进度记录与TODO

### 进度记录

- [x] 2019/5/29 client_golang 代码阅读
- [x] 2019/6/10 初步整理完client_golang sdk 中关于指标数据收集的笔记
- [x] 2019/6/16 整理完Metric 相关代码文档
  - [x] vec 指标的代码实现与笔记
  - [x] gauge, summary 等指标的使用与实现
- [x] api 相关代码逻辑实现 2019/6/23

### TODO

转到server/main

- [ ] prometheus 主程序逻辑
- [ ] prometheus scrape 代码实现
- [ ] prometheus 服务发现代码实现
- [ ] web
