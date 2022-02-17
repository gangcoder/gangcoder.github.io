---
title: nsq structure
date: 2018-09-11 14:12:17
category: language, go, nsq
---

1. 代码情况
2. nsq 源码目录
3. go-nsq 源码目录结构

介绍代码情况，了解代码组织方式，这样大家如果想自己深入源码，也知道从哪里入手

## 1. 代码情况

代码地址

- [nsq 源代码 https://github.com/nsqio/nsq](https://github.com/nsqio/nsq)
- [go 客户端库 https://github.com/nsqio/go-nsq](https://github.com/nsqio/go-nsq)

nsq 代码量

```sh
--------------------------------------------------------------------------------
Language                      files          blank        comment           code
--------------------------------------------------------------------------------
Go                              109           3040            569          16188
--------------------------------------------------------------------------------
```

go-nsq 代码量

```
-------------------------------------------------------------------------------
Language                     files          blank        comment           code
-------------------------------------------------------------------------------
Go                              19            697            595           3710
-------------------------------------------------------------------------------
```

## 2. nsq 源码目录

nsq 源代码

- apps 应用
- nsqd 代码
- lookupd 代码
- 公共代码，辅助函数
- 控制台代码


```sh
nsq
├── apps //应用
│   ├── nsqadmin //控制台
│   ├── nsqd //nsqd 主程序
│   ├── nsqlookupd //lookupd 程序
│   ├── nsq_tail //将topic 数据导出到终端
│   └── to_nsq //将终端数据写入nsq
├── internal //公共代码
├── nsqadmin //控制台
├── nsqd //nsqd 代码实现
│   ├── channel.go //channel 操作代码
│   ├── client_v2.go //客户端与服务端的连接
│   ├── http.go //nsqd http 接口处理代码
│   ├── lookup.go //nsqd 与lookup 之前的通信处理
│   ├── lookup_peer.go //nsqd 与lookup 的连接处理
│   ├── message.go //msg 处理
│   ├── nsqd.go //nsqd 主逻辑
│   ├── protocol_v2.go //tcp 接口处理代码
│   ├── tcp.go //tcp handler
│   ├── topic.go //topic 处理代码
└── nsqlookupd //lookup 实现
    ├── http.go //lookup 的http 处理接口
    ├── lookup_protocol_v1.go //lookup 的tcp 接口逻辑处理代码
    ├── nsqlookupd.go //lookup 主逻辑
    ├── registration_db.go //定义数据结构
    └── tcp.go //tcp 监听处理handler
```

## 3. go-nsq 源码目录结构

nsq 的客户端golang 库

- consumer 消费者
- producer 生产者

```sh
go-nsq
├── api_request.go //http 客户端代码， 用于消费者从lookupd 发现nsqd 地址
├── command.go //客户端命令封装
├── conn.go //客户端与服务端连接处理
├── consumer.go //消费者接入
├── delegates.go //委托层代码
├── message.go //msg 消息体初始化和处理
├── producer.go //消息发布
└── protocol.go //数据序列化好处理
```


## 参考资料

- [官方文档 http://nsq.io/overview/design.html](http://nsq.io/overview/design.html)