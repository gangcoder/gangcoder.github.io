<!-- ---
title: consul code
date: 2019-07-23 21:28:31
category: src, consul
--- -->

consul code


## 代码目录结构

代码库地址是`github.com/hashicorp/consul` , consul 的所有命令包括客户端Agent 和服务端都在一个程序包中。

```
├── agent // agent 相关功能，包括agent 客户端和consul 服务端功能
    └── consul // consul 服务端相关功能
├── api // 用于客户端调用的api 封装
├── command // consul 命令实现
├── connect
├── main.go // 入口函数
├── snapshot
├── ui
├── ui-v2
├── watch
└── website
```
