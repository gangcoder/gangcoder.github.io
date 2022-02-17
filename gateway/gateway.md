<!-- ---
title: api gateway
date: 2018-11-23 20:51:00
category: src, gateway
--- -->

api gateway

## 内容安排

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

## 学习记录

2020/05/27
1. 文档阅读
2. 源码概览

2020/05/28
1. tyk 文档 https://tyk.io/docs/basic-config-and-security/
2. tyk 源码概览

2020/05/29
1. goku 官方文档 https://help.eolinker.com/#/tutorial/?groupID=c-307&productID=19
2. goku 源码概览

2020/07/06
1. https://github.com/containous/traefik 了解
2. https://github.com/bfenetworks/bfe/ 了解
3. 整理网关源码学习路径

2020/07/11
- traefix 文档整理
- traefix 使用
- traefix 源码整理

traefik 中文文档: https://www.bookstack.cn/read/traefik/0.md

traefik 官方文档: https://docs.traefik.io/

https://containo.us/traefik/

https://www.bfe-networks.net/zh_cn/ABOUT/


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

## 步骤

1. 文档阅读
   1. tyk 官方文档 https://tyk.io/docs/
   2. 文档 https://github.com/TykTechnologies/tyk-docs
2. tyk 源码 https://github.com/TykTechnologies/tyk.git
   1. 终端工具 https://github.com/TykTechnologies/tyk-cli.git
3. 其他参考网关
   1. https://github.com/containous/traefik 29.5 
   2. https://github.com/bfenetworks/bfe/
   3. https://github.com/eolinker/goku-api-gateway 1.9k
   4. goku 官方文档 https://help.eolinker.com/#/tutorial/?groupID=c-307&productID=19
   5. https://github.com/fagongzi/manba 2.5k 

https://s0docs0traefik0io.icopy.site/routing/entrypoints/

## 参考资料

1. https://github.com/containous/traefik 29.5k
2. https://github.com/TykTechnologies/tyk 5.6k
3. https://github.com/bfenetworks/bfe/ 3.7k
4. https://github.com/apache/incubator-apisix 2.7k
5. https://github.com/fagongzi/manba 2.5k
6. https://github.com/eolinker/goku-api-gateway 2k


https://github.com/topics/api-gateway?l=go
https://github.com/TykTechnologies/tyk
https://github.com/yosriady/api-development-tools#api-gateways

27.8K
https://github.com/containous/traefik
https://docs.traefik.cn/basics
https://docs.traefik.io/


gateway
https://github.com/search?o=desc&q=gateway&s=stars&type=Repositories


gateway language:golang
https://github.com/search?q=gateway+language%3Agolang&type=Repositories


api 相关的有用资源
https://github.com/yosriady/api-development-tools#api-gateways

## 学习安排

https://github.com/TykTechnologies/tyk 4254
api 网关，有商业版软件，推荐深入学习


https://github.com/fagongzi/gateway 1565
api 网关，国内网关，推荐参考学习


https://github.com/hellofresh/janus 1611
api 网关，可供学习


https://github.com/eolinker/EOLINKER-GoKu-API-Gateway-CE
国内开源gateway，可以参考学习


https://github.com/devopsfaith/krakend 1293
基于中间件的网关，参考学习

https://github.com/grpc-ecosystem/grpc-gateway
protocal buffer 插件，将数据转为json
可供参考学习


## 参考资料

- [](https://github.com/fagongzi/gateway)

https://mp.weixin.qq.com/s/2X7hbEYdOVGFxOA226JFjg

网关参考
https://www.oschina.net/p/apioak

apisix
https://space.bilibili.com/551921247
