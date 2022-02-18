<!-- ---
title: nsq features
date: 2018-08-31 17:04:28
category: language, go, nsq
--- -->

nsq features

## Delegate

delegate 使用

```go
NewConn(addr, &r.config, &consumerConnDelegate{r})
```

### Conn 代理

```go
// ConnDelegate
type ConnDelegate interface {}

//consumer conn delegate 实现
type consumerConnDelegate struct {
	r *Consumer
}

//producer conn delegate 实现
type producerConnDelegate struct {
	w *Producer
}
```

### msg 代理

```go
// MessageDelegate interface
type MessageDelegate interface {}

//msg 代理实现
type connMessageDelegate struct {
	c *Conn
}
```



## 参考资料

- [Delegate](#delegate)
    - [Conn 代理](#conn-%E4%BB%A3%E7%90%86)
    - [msg 代理](#msg-%E4%BB%A3%E7%90%86)
- [参考资料](#%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99)