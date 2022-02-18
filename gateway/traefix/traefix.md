<!-- ---
title: Traefix 网关
date: 2022-02-18 09:48:58
category: showcode, gateway, traefix
--- -->

# Traefix 网关


## 内容

* [Traefix 介绍](/gateway/traefix/traefix_introduce.md)
* [Traefix 安装与启动](/gateway/traefix/traefix_startup.md)
* [Traefix 配置](/gateway/traefix/traefix_config.md)
* [Traefix 服务入口](/gateway/traefix/traefix_main.md)
* [Traefix Provider 实现](/gateway/traefix/traefix_provider.md)
* [Traefix 配置监听](/gateway/traefix/traefix_watcher.md)
* [Traefix 管理创建器实现](/gateway/traefix/traefix_internal_manager.md)
* [Traefix 负载均衡实现](/gateway/traefix/traefix_http_manager.md)
* [Traefix EntryPoints](/gateway/traefix/traefix_entrypoints.md)
* [Traefix TCP router](/gateway/traefix/traefix_tcp_router.md)
* [Traefix 路由创建器实现](/gateway/traefix/traefix_routerfactory.md)
* [Traefix RouterManager](/gateway/traefix/traefix_routermanager.md)
* [Traefix TCP RouterManager](/gateway/traefix/traefix_routermanager_tcp.md)
* [Traefix Http 规则路由器](/gateway/traefix/traefix_rulerouter.md)
* [Traefix Balancer](/gateway/traefix/traefix_balancer.md)

traefix 文档
1. traefix 介绍
2. traefix 概念
3. traefix 安装与启动
   1. 控制台
   2. 日志
   3. 指标
4. traefix 配置
   1. 服务发现
   2. 路由与服务
   3. 中间件

traefix 源码
1. main 服务入口
2. 启动服务
   1. 服务启动逻辑
   2. s.tcpEntryPoints.Start()
   3. s.udpEntryPoints.Start()
   4. s.watcher.Start()
3. provider 实现
4. 配置监听实现
5. EntryPoints 入口点实现，包括http, tcp, udp
6. managerFactory
7. routerFactory
8. chainBuilder
9.  metricsRegistry
10. tcpEntryPoints.Start 实现


## 记录

2020/07/11
- traefix 文档整理
- traefix 使用
- traefix 源码整理

traefik 中文文档: https://www.bookstack.cn/read/traefik/0.md

traefik 官方文档: https://docs.traefik.io/

https://containo.us/traefik/


## 参考资料

- []()