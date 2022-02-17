---
title: golang topic
date: 2018-08-20 21:22:33
category: language, go, nsq
---

golang topic

topic 消息投递

1. 获取topic
2. 创建消息
3. 消息发布到topic

## 发布消息

```go
func (t *Topic) PutMessage(m *Message) error {
	err := t.put(m)
	atomic.AddUint64(&t.messageCount, 1)
	return nil
}
```

消息写入内存或者备用存储中

```go
func (t *Topic) put(m *Message) error {
	select {
	case t.memoryMsgChan <- m: //t.memoryMsgChan: make(chan *Message, ctx.nsqd.getOpts().MemQueueSize)
	default:
		b := bufferPoolGet()
		err := writeMessageToBackend(b, m, t.backend)
		bufferPoolPut(b)
	}
	return nil
}
```


## 参考资料

- [发布消息](#发布消息)
- [参考资料](#参考资料)