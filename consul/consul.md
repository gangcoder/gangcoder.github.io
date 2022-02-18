<!-- ---
title: consul
date: 2018-11-15 20:39:30
category: src, consul
--- -->

consul

## 内容目录

* [Consul 使用指南](/consul/doc/consul_guide.md)
* [Consul 安装与启动](/consul/doc/consul_startup.md)
* [Consul 组件结构](/consul/consul_structure.md)
* [Consul 代码结构](/consul/consul_code.md)
* [Consul 入口函数](/consul/consul_main.md)
* [Consul version 命令](/consul/consul_version.md)
* [Consul KV 命令](/consul/consul_command_kv.md)
* [Consul agent 启动](/consul/consul_agent.md)
* [Consul agent client 模式](/consul/consul_agent_client.md)
* [Consul agent server 模式](/consul/consul_agent_server.md)
* [Consul 服务处理逻辑](/consul/consul_server.md)
* [Consul 服务端KV 逻辑](/consul/consul_server_kv.md)
* [Consul API 客户端](/consul/consul_api.md)
* [Consul Http 服务](/consul/consul_http.md)
* [Consul Http 服务KV 实现](/consul/consul_http_kv.md)

## 参考资料

Consul 简介
https://www.cnblogs.com/niejunlei/p/6514859.html

Consul 架构（译）
https://www.cnblogs.com/niejunlei/p/8350861.html

Consul 启动命令，Web UI
https://www.cnblogs.com/niejunlei/p/5982911.html

Consul 服务发现和配置
https://www.cnblogs.com/niejunlei/p/5981963.html

https://www.bilibili.com/video/BV1pT4y1374Z

- [x] [中文Consul 使用手册](http://www.liangxiansen.cn/2017/04/06/consul/)
- [x][中文官方文档翻译（不全）](https://bishion.github.io/2018/07/28/consul-tutorial/#%E6%8C%87%E5%8D%97)
- [consul 入门指南](https://book-consul-guide.vnzmi.com/01_what_is_consul.html)
- [官方文档](https://www.consul.io/docs/install/index.html)

- [github](https://github.com/hashicorp/consul.git)

consul 学习官方文档
https://learn.hashicorp.com/consul

http://consul.la/
http://www.liangxiansen.cn/2017/04/06/consul/
https://legacy.gitbook.com/book/vincentmi/consul-guide/details

https://bishion.github.io/2018/07/28/consul-tutorial

consul
https://www.consul.io/use-cases/service-discovery-and-health-checking

acl
https://learn.hashicorp.com/consul/security-networking/production-acls

https://www.consul.io/use-cases/network-infrastructure-automation

https://learn.hashicorp.com/consul?track=getting-started#getting-started

0719
https://bishion.github.io/2018/07/28/consul-tutorial/#%E4%BB%8B%E7%BB%8D

## 学习记录

- 2019/07/13 学习：根据文档了解基本使用方法，下载源代码
- 2019/07/22 学习完consul 中文使用手册和中文官方文档翻译；程序启动和使用命令
- 2020/06/18 consul 代码阅读整理
- 2020/06/23 consul http 服务整理
- 2020/06/28 consul server rpc 服务整理


## 内容计划

1. 阅读中文文档
   1. 中文使用文档
   2. 中文官方文档
2. 搭建最简软件使用环境
   1. 客户端使用
   2. 服务端使用
   3. 熟悉软件使用，保留使用文档
3. 搭建软件编译环境，下载依赖


## 学习安排

1. 阅读文档
   1. [x] 中文使用文档和中文官方文档
2. 搭建最简软件使用环境命令
   1. [x] 客户端使用
   2. [x] 服务端使用
   3. [x] 熟悉软件使用，保留使用文档
3. 搭建软件编译环境，下载依赖
   1. [ ] 源码编译
4. 阅读代码
   1. [ ] main 启动
   2. [ ] client 命令
   3. [ ] server 命令
