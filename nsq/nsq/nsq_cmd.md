---
title: nsq command
date: 2018-08-27 12:44:57
category: language, go, nsq
---

nsq command

```go
//Command 发往nsqd 的命令
type Command struct {
	Name   []byte
	Params [][]byte
	Body   []byte
}
```

```go
// Publish 创建一条 PUB cmd
func Publish(topic string, body []byte) *Command {
	var params = [][]byte{[]byte(topic)}
	return &Command{[]byte("PUB"), params, body}
}
```

## 参考资料

- [参考资料](#%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99)