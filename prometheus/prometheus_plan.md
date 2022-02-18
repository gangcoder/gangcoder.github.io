<!-- ---
title: prometheus plan
date: 2019-05-16 13:00:03
category: src, prometheus
--- -->

prometheus plan


## 目标

掌握prometheus 整体设计思路和代码实现，从整体理解prometheus，并且掌握设计思路和实现技巧。

重点是掌握设计思想和实现方式，避免钻入某个特定细节陷入死角。

## 源码仓库

https://github.com/prometheus

```
https://github.com/prometheus/prometheus.git
https://github.com/prometheus/alertmanager.git
https://github.com/prometheus/common.git
https://github.com/prometheus/tsdb.git
https://github.com/prometheus/pushgateway.git
https://github.com/prometheus/client_golang.git
https://github.com/prometheus/promu.git
```

## 资料

- [https://prometheus.io/](https://prometheus.io/)
- [Prometheus 系统监控方案 一](https://www.cnblogs.com/vovlie/p/Prometheus_CONCEPTS.html)

中文文档
https://www.bookstack.cn/read/prometheus-manual/README.md
https://github.com/1046102779/prometheus


## 学习安排

重点是学习设计思路，最后的内容整理


- 文档
 - 中文文档
 - 英文文档
- 代码结构
  - 代码笔记

代码部分

1. 安装与启动
2. 程序入口：包括服务端，客户端
3. 程序架构与实现
4. 实现细节与步骤

2019/12/22
- prom 指南阅读