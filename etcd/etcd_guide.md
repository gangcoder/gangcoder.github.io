---
title: etcd 指南
date: 2019-08-04 01:09:07
category: showcode, etcd
---

# etcd 指南

"etcd"这个名字源于两个想法，即　unix "/etc" 文件夹和分布式系统"d"istibuted。 "/etc" 文件夹为单个系统存储配置数据的地方，而 etcd 存储大规模分布式系统的配置信息。因此，"d"istibuted　的 "/etc" ，是为 "etcd"。

etcd 是一个分布式键值对存储，设计用来可靠而快速的保存关键数据并提供访问。分布式系统使用 etcd 作为一致性键值存储，用于配置管理，服务发现和协调分布式工作。

## 术语

- Node / 节点：raft 状态机的一个实例。
- Member / 成员：etcd 的一个实例。它承载一个 node/节点，并为client/客户端提供服务。
- Cluster / 集群：由多个　member/成员组成。每个成员的节点遵循 raft 一致性协议来复制日志。
- Peer / 同伴：同一个集群中的其他成员。
- Proposal / 提议：一个需要完成　raft　协议的请求(例如写请求，配置修改请求)。
- Client / 客户端：集群 API的调用者。



## 参考资料

- []()

