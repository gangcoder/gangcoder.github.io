<!-- ---
title: nsq structure
date: 2018-09-11 14:12:17
category: language, go, nsq
--- -->

nsq 代码情况，包括目录结构


## 1. 代码情况

资源文档

- [源代码库 https://github.com/nsqio/nsq](https://github.com/nsqio/nsq)
- [go 客户端库 https://github.com/nsqio/go-nsq](https://github.com/nsqio/go-nsq)

代码量

```sh
--------------------------------------------------------------------------------
Language                      files          blank        comment           code
--------------------------------------------------------------------------------
Go                              109           3040            569          16188
--------------------------------------------------------------------------------
```

## nsq 源码目录

nsq 源代码

nsq

- 应用bin
- nsqd 代码实现
- lookup 实现
- 内核公共代码
- 后台控制台


```sh
nsq
├── apps //应用bin
│   ├── nsqadmin
│   ├── nsqd
│   ├── nsqlookupd
│   ├── nsq_stat
│   ├── nsq_tail
│   ├── nsq_to_file
│   ├── nsq_to_http
│   ├── nsq_to_nsq
│   └── to_nsq
├── internal //内核公共代码
│   ├── app
│   ├── auth
│   ├── clusterinfo
│   ├── dirlock
│   ├── http_api
│   ├── lg
│   ├── pqueue
│   ├── protocol
│   ├── quantile
│   ├── statsd
│   ├── stringy
│   ├── test
│   ├── util
│   ├── version
│   └── writers
├── nsqadmin //控制台
│   ├── bindata.go
│   ├── context.go
│   ├── gulp
│   ├── gulpfile.js
│   ├── http.go
│   ├── logger.go
│   ├── notify.go
│   ├── nsqadmin.go
│   ├── options.go
│   ├── package.json
│   ├── package-lock.json
│   ├── README.md
│   ├── static
│   └── test
├── nsqd //nsqd 代码实现
│   ├── backend_queue.go
│   ├── buffer_pool.go
│   ├── channel.go
│   ├── client_v2.go
│   ├── context.go
│   ├── dqname.go
│   ├── dqname_windows.go
│   ├── dummy_backend_queue.go
│   ├── guid.go
│   ├── http.go
│   ├── in_flight_pqueue.go
│   ├── logger.go
│   ├── lookup.go
│   ├── lookup_peer.go
│   ├── message.go
│   ├── nsqd.go
│   ├── options.go
│   ├── protocol_v2.go
│   ├── README.md
│   ├── statsd.go
│   ├── stats.go
│   ├── tcp.go
│   ├── test
│   ├── topic.go
└── nsqlookupd //lookup 实现
    ├── client_v1.go
    ├── context.go
    ├── http.go
    ├── logger.go
    ├── lookup_protocol_v1.go
    ├── nsqlookupd.go
    ├── options.go
    ├── README.md
    ├── registration_db.go
    └── tcp.go
```

### go-nsq 源码目录结构

golang 的nsq 客户端库源码

- consumer
- producer

```sh
go-nsq
├── api_request.go
├── command.go
├── config_flag.go
├── config.go
├── conn.go
├── consumer.go
├── delegates.go
├── errors.go
├── message.go
├── producer.go
├── protocol.go
├── states.go
└── version.go
```


## 参考资料

- [官方文档 http://nsq.io/overview/design.html](http://nsq.io/overview/design.html)