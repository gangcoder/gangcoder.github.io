<!-- ---
title: prometheus
date: 2018-11-17 06:20:48
category: src, prometheus
--- -->

prometheus


## 内容

学习内容清单与学习记录

### 软件功能部分

- [安装](/prometheus/prometheus_install.md)
- [查询](/prometheus/prometheus_querying.md)
- [术语](/prometheus/prometheus_glossary.md)

中文文档

- 常见问题 https://www.bookstack.cn/read/prometheus-manual/introduction-questions.md
- 路线图
- https://www.bookstack.cn/read/prometheus-manual/concepts-data_model.md
- https://www.bookstack.cn/read/prometheus-manual/concepts-metric_types.md
- https://www.bookstack.cn/read/prometheus-manual/prometheus-querying-operators.md
- https://www.bookstack.cn/read/prometheus-manual/prometheus-querying-query_examples.md
- https://www.bookstack.cn/read/prometheus-manual/prometheus-querying-http_api.md

https://jeremyxu2010.github.io/2018/08/%E7%A0%94%E7%A9%B6%E7%9B%91%E6%8E%A7%E7%B3%BB%E7%BB%9F%E4%B9%8Bprometheus/

内容完善 ***
https://www.ctolib.com/docs/sfile/prometheus-book/promql/prometheus-metrics-types.html

操作指南
https://github.com/yunlzheng/prometheus-book

### 代码内容部分

1. 代码实现；
2. 软件结构逻辑；
3. 各组件之间的交互逻辑；

代码阅读与功能介绍同步进行，先介绍有什么功能，功能是什么样的，在附上代码实现细节。

编写顺序：

1. 软件前后端整体架构是什么样的，简单介绍即可，为第2 步和第3 步开启论述：prometheus 是CS 架构，客户端产生数据，服务端拉取数据
2. 客户端数据是什么样的，怎产生，怎么存储，怎么暴露给服务端拉取
3. prometheus 服务端是怎么拉取数据的，怎么解析配置文件，怎么启动goroutine 去拉取数据，拉取的数据怎么存，这些功能的具体代码实现
4. 数据是怎么存储的，先简单介绍有哪些interface，不需要深入细节，避免纠缠在小处
5. 数据有了后怎么运行告警规则，解析数据，怎么发送到告警中心，这些代码实现
6. 告警中心的代码实现，怎么接收数据，怎么去重处理，怎么发送告警，等功能的处理逻辑，交互方式和代码实现
7. 回过头讲解，对于pushgateway 怎么处理，怎么主动推荐， 怎么暴露端口
8. 穿插讲解 common 通用函数的讲解，以及实现
9. 对于数据存储 tsdb 进行针对性就介绍，包括1. 代码实现；2. 软件结构逻辑；3. 内部各组件之间的交互逻辑；


- [client_golang]()
  - api
  - prometheus
- prometheus
  - scrape
  - notify
  - rules
- alertmanager
  - cmd
  - api
  - provider
  - store
  - dispatch
  - silence
  - inhibit
  - notify
  - cli
  - client
- pushgateway
  - handler
  - storage
- common
- promu
- tsdb

## TODO 与重点突破

Histogram 和 summary

https://www.ctolib.com/docs/sfile/prometheus-book/promql/prometheus-metrics-types.html
http://ylzheng.com/2017/07/07/prometheus-exporter-example-go/
https://blog.frognew.com/2017/05/prometheus-intro.html
https://blog.frognew.com/2018/01/2017-summary.html

