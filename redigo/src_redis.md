---
title: go redis类库源码阅读
date: 2019-01-21 21:28:15
category: src, redis
---

go redis类库源码解析

golang redis 类库


## Connections

Conn 是redis 的主要 interface，Conn 通过 `Dial`, `DialWithTimeout`, `NewConn` 创建。

在App 退出时，需要调用 `Close` 关闭 Conn


## 执行redis 命令

```go
Do(commandName string, args ...interface{}) (reply interface{}, err error)
```

## Pipeline

- Send 将cmd 写入 Conn output buffer. 
- Flush 将 Conn output buffer 写入服务端
- Receive 读取一条服务端响应数据


## 响应处理函数

Bool, Int, Bytes, String, Strings and Values 函数可以将redis 响应结果转为golang 数据类型。

```go
exists, err := redis.Bool(c.Do("EXISTS", "foo"))
if err != nil {
    // handle error return from c.Do or type conversion error.
}
```

## 连接池


## scan



## 验证与tls


## 订阅




## 参考资料

- [github.com/gomodule/redigo](github.com/gomodule/redigo)

