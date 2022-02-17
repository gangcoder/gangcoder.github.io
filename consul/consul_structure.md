<!-- ---
title: consul 结构图
date: 2019-07-29 00:58:22
category: showcode, consul
--- -->

# consul 结构图

Consul 是一个分布式高可以的系统，每个实例节点都运行一个Consul agent 客户端。

Agent 与一个或多个Consul Server 进行交互，Consul Server 用于存放和复制数据。

完成Consul的安装后，必须运行agent。agent可以运行为server或client模式。每个数据中心至少必须拥有一台server。其他的agent运行为client模式，一个client是一个非常轻量级的进程.用于注册服务,运行健康检查和转发对server的查询。agent必须在集群中的每个主机上运行。

## 应用与Consul 的调用逻辑

客户端，consul agent 客户端和consul agent 服务端之间的调用逻辑：

1. 客户端通过http 请求 consul agent 客户端
2. consul agent 客户端将请求转发给consul agent 服务端


## 服务注册与发现

应用A 向Consul 注册服务（比如 api 或者mysql），其他应用（如应用B）可以使用Consul 发现应用A 的具体提供者。

## 健康检查

Consul 可以配置健康检查，检查指定服务(如：WebServer）是否返回了200 OK 状态码。

## kv 存储

应用可以使用Consul 存储Key/Value 配置，比如动态配置,功能标记。

## 事件

Consul 事件处理

## http 服务

## grpc 服务

## DNS 服务


## 集群与多数据中心

Consul 支持开箱即用的多数据中心

