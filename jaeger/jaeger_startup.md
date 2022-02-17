<!-- ---
title: jaeger 安装与启动
date: 2019-08-07 15:01:57
category: showcode, jaeger
--- -->

# jaeger 安装与启动

## 安装

下载地址: https://www.jaegertracing.io/download/

下载可执行文件使用即可。

- example-hotrod 生成trace 的测试工具
- jaeger-agent agent 程序，部署在每台机器上
- jaeger-all-in-one 一键启动程序，包含UI 界面，collector，query，agent 和in-memory storage。
- jaeger-collector 收集器
- jaeger-query 查询器

## 启动

### jaeger-all-in-one

```
jaeger-all-in-one
```

### example-hotrod

hotrod 测试工具启动:

```
example-hotrod all
```

## 参考资料

- [https://github.com/jaegertracing/jaeger/tree/master/examples/hotrod](https://github.com/jaegertracing/jaeger/tree/master/examples/hotrod)
- [https://medium.com/opentracing/take-opentracing-for-a-hotrod-ride-f6e3141f7941](https://medium.com/opentracing/take-opentracing-for-a-hotrod-ride-f6e3141f7941)

