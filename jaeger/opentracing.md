<!-- ---
title: opentrace
date: 2019-06-30 02:01:10
category: src, jaeger
--- -->

# opentrace 简介

## opentracing 作用

调用链追踪最先由googel 在Dapper 这篇论文中提出，OpenTracing 主要定义了相关的协议以及接口，这样各个语言只要按照Opentracing 的接口以及协议实现数据上报，那么调用信息就能统一被收集。

OpenTracing 是一个轻量级的标准化层，它位于应用程序/类库和追踪或日志分析程序之间。

```
+-------------+  +---------+  +----------+  +------------+
| Application |  | Library |  |   OSS    |  |  RPC/IPC   |
|    Code     |  |  Code   |  | Services |  | Frameworks |
+-------------+  +---------+  +----------+  +------------+
       |              |             |             |
       |              |             |             |
       v              v             v             v
  +------------------------------------------------------+
  |                     OpenTracing                      |
  +------------------------------------------------------+
     |                |                |               |
     |                |                |               |
     v                v                v               v
+-----------+  +-------------+  +-------------+  +-----------+
|  Tracing  |  |   Logging   |  |   Metrics   |  |  Tracing  |
| System A  |  | Framework B |  | Framework C |  | System D  |
+-----------+  +-------------+  +-------------+  +-----------+
```

## 参考资料

- opentracing 英文文档 https://opentracing.io/guides/golang/
- opentracing 中文 https://github.com/opentracing-contrib/opentracing-specification-zh
- opentracing 中文 https://opentracing-contrib.github.io/opentracing-specification-zh/specification.html
- opentracing 中文网站 https://wu-sheng.gitbooks.io/opentracing-io/content/
