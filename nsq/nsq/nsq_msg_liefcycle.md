---
title: nsq msg
date: 2018-08-30 10:47:40
category: language, go, nsq
---

nsq msg


## 1. producer 发布msg

### 1.1 msg 写入chan

```go
// Publish 发布msg 到topic
func (w *Producer) Publish(topic string, body []byte) error {
	return w.sendCommand(Publish(topic, body))
}

// Publish 创建一条 PUB cmd
func Publish(topic string, body []byte) *Command {
	var params = [][]byte{[]byte(topic)}
	return &Command{[]byte("PUB"), params, body}
}

//异步发布消息
func (w *Producer) sendCommand(cmd *Command) error {
	doneChan := make(chan *ProducerTransaction)
	err := w.sendCommandAsync(cmd, doneChan, nil)
}

//将待发布的消息写入chan，由goroutine 异步写入nsqd
func (w *Producer) sendCommandAsync(cmd *Command, doneChan chan *ProducerTransaction, args []interface{})
    //连接nsqd 并且异步处理数据投递逻辑
    err := w.connect()

	t := &ProducerTransaction{
		cmd:      cmd,
		doneChan: doneChan,
		Args:     args,
	}

	w.transactionChan <- t:
}
```

### 1.2 连接nsqd 发布msg

```go
func (w *Producer) connect() error {
	//建立与nsqd 的连接
	w.conn = NewConn(w.addr, &w.config, &producerConnDelegate{w})
	_, err := w.conn.Connect()
    
    //消息逻辑处理
	go w.router()
}

//路由执行读取和写入操作
func (w *Producer) router() {
	for {
		select {
		case t := <-w.transactionChan: //消息异步投递
			w.transactions = append(w.transactions, t)
			err := w.conn.WriteCommand(t.cmd)
		case data := <-w.responseChan: //消息投递结果异步处理
			w.popTransaction(FrameTypeResponse, data)
        }
    }
}
```

## 2. nsqd 投递msg

### 2.1 读取msg

```go
//循环读取tcp 连接请求数据，然后处理
func (p *protocolV2) IOLoop(conn net.Conn) error {
	clientID := atomic.AddInt64(&p.ctx.nsqd.clientIDSequence, 1)
	client := newClientV2(clientID, conn, p.ctx)

	//处理将消息推送到consumer 的任务
	messagePumpStartedChan := make(chan bool)
	go p.messagePump(client, messagePumpStartedChan)
	<-messagePumpStartedChan

	for {
		//读取tcp 连接接收到的数据
		line, err = client.Reader.ReadSlice('\n')
		params := bytes.Split(line, separatorBytes)

		//处理读取到的数据
		var response []byte
		response, err = p.Exec(client, params)

		//返回处理结果
		if response != nil {
			err = p.Send(client, frameTypeResponse, response)
		}
	}

	//结束处理
	conn.Close()
	close(client.ExitChan)
	return err
}
```

处理具体请求逻辑，将各类请求分发到具体函数上

```go
func (p *protocolV2) Exec(client *clientV2, params [][]byte) ([]byte, error) {
	if bytes.Equal(params[0], []byte("IDENTIFY")) {
		return p.IDENTIFY(client, params)
	}
	
	switch {
	case bytes.Equal(params[0], []byte("RDY")):
		return p.RDY(client, params)
	case bytes.Equal(params[0], []byte("PUB")):
		return p.PUB(client, params)
	}
}
```

处理接受到的投递消息

```go
func (p *protocolV2) PUB(client *clientV2, params [][]byte) ([]byte, error) {
	//获取topic 名称
	topicName := string(params[1])

	//读取message 信息
	messageBody := make([]byte, bodyLen)
	_, err = io.ReadFull(client.Reader, messageBody)

	//获取topic 实体
	topic := p.ctx.nsqd.GetTopic(topicName)

	//创建消息体
	msg := NewMessage(topic.GenerateID(), messageBody)
	//消息投入到topic
	err = topic.PutMessage(msg)
	
	return okBytes, nil
}
```

将消息发布到topic chan 上

```go
func (t *Topic) PutMessage(m *Message) error {
	err := t.put(m)
	atomic.AddUint64(&t.messageCount, 1)
	return nil
}

//消息写入内存或者备用存储中
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

### 2.2 msg 转推给consumer

1. 取出msg
2. 发送给consumer

```go
func (p *protocolV2) messagePump(client *clientV2, startedChan chan bool) {
	var memoryMsgChan chan *Message
	var backendMsgChan chan []byte
	var subChannel *Channel

	for {
		//获取消息
		memoryMsgChan = subChannel.memoryMsgChan
		backendMsgChan = subChannel.backend.ReadChan()
		flusherChan = nil
		
		select {
		case <-flusherChan:
			//flush 消息
			client.writeLock.Lock()
			err = client.Flush()
			client.writeLock.Unlock()
			flushed = true
		case <-client.ReadyStateChan:
		case subChannel = <-subEventChan:
		case b := <-backendMsgChan:
			//持久存储的数据
			msg, err := decodeMessage(b)

			subChannel.StartInFlightTimeout(msg, client.ID, msgTimeout)
			client.SendingMessage()
			err = p.SendMessage(client, msg)
		case msg := <-memoryMsgChan:
			msg.Attempts++

			subChannel.StartInFlightTimeout(msg, client.ID, msgTimeout)
			client.SendingMessage()
			err = p.SendMessage(client, msg)
		}
	}
}
```

发送消息

```go
func (p *protocolV2) SendMessage(client *clientV2, msg *Message) error {
	var buf = &bytes.Buffer{}
	_, err := msg.WriteTo(buf)

	err = p.Send(client, frameTypeMessage, buf.Bytes())
}

func (p *protocolV2) Send(client *clientV2, frameType int32, data []byte) error {
	_, err := protocol.SendFramedResponse(client.Writer, frameType, data)
}

func SendFramedResponse(w io.Writer, frameType int32, data []byte) (int, error) {
	beBuf := make([]byte, 4)
	n, err := w.Write(beBuf)

	binary.BigEndian.PutUint32(beBuf, uint32(frameType))
	n, err = w.Write(beBuf)

	n, err = w.Write(data)
	return n + 8, err
}
```


## 3. consumer 消费

1. 接收msg 进行处理
2. 订阅topic
3. 通知RDY

### 3.1 接收msg 进行处理逻辑

1. 编写msg handler
2. 添加消息 handler
3. 开启消息 handler loop

```go
// AddHandler 添加handler
func (r *Consumer) AddHandler(handler Handler) {
	r.AddConcurrentHandlers(handler, 1)
}

// 添加handler
func (r *Consumer) AddConcurrentHandlers(handler Handler, concurrency int) {
	go r.handlerLoop(handler)
}

//从 incomingMessages 中取出msg 进行处理
func (r *Consumer) handlerLoop(handler Handler) {
	for {
		//从chan 中取出msg
		message, ok := <-r.incomingMessages

		//消费msg
		err := handler.HandleMessage(message)
		if err != nil {
			if !message.IsAutoResponseDisabled() {
				//requeue
				message.Requeue(-1)
			}
		}

		//msg 处理完成
		if !message.IsAutoResponseDisabled() {
			message.Finish()
		}
	}
}
```

### 3.2 与nsqd　建立连接


连接nsqd 处理逻辑

```go
//连接到nsqd
func (r *Consumer) ConnectToNSQD(addr string) error {
	//连接
	conn := NewConn(addr, &r.config, &consumerConnDelegate{r})
    
    //记录连接
	_, pendingOk := r.pendingConnections[addr]	
    r.nsqdTCPAddrs = append(r.nsqdTCPAddrs, addr)
	
	//连接到nsqd 并且进行认证
	resp, err := conn.Connect()
    
    //订阅topic, channel
	cmd := Subscribe(r.topic, r.channel)
	err = conn.WriteCommand(cmd)
	
    //发送RDY 消息，开始接收数据
    r.maybeUpdateRDY(c)
}
```

### 3.３ 订阅topic


订阅topic

```go
// Subscribe 订阅topic 的cmd
func Subscribe(topic string, channel string) *Command {
	var params = [][]byte{[]byte(topic), []byte(channel)}
	return &Command{[]byte("SUB"), params, nil}
}
```


### 3.４ 通知RDY

```go
func (r *Consumer) maybeUpdateRDY(conn *Conn) {
    r.updateRDY(conn, count)
}

func (r *Consumer) updateRDY(c *Conn, count int64) error {
    r.sendRDY(c, count)
}


func (r *Consumer) sendRDY(c *Conn, count int64) error {
	err := c.WriteCommand(Ready(int(count)))
}
```

### 3.5 通过lookup 获取nsqds


1. 通过lookup 查询nsqd
2. 连接nsqd

```go
// ConnectToNSQLookupd 添加lookupd 地址
func (r *Consumer) ConnectToNSQLookupd(addr string) error {
	r.lookupdHTTPAddrs = append(r.lookupdHTTPAddrs, addr)
	numLookupd := len(r.lookupdHTTPAddrs)

	// 处理第一个lookup 地址时开启goroutine
	if numLookupd == 1 {
		r.queryLookupd()
	}
}

// 查询lookup，获取producer
func (r *Consumer) queryLookupd() {
	//查询lookupd
	endpoint := r.nextLookupdEndpoint()
	err := apiRequestNegotiateV1("GET", endpoint, nil, &data)
	
	//解析nsqd 数据
	for _, producer := range data.Producers {
		broadcastAddress := producer.BroadcastAddress
		port := producer.TCPPort
		joined := net.JoinHostPort(broadcastAddress, strconv.Itoa(port))
		nsqdAddrs = append(nsqdAddrs, joined)
	}
	
	//连接nsqd
	for _, addr := range nsqdAddrs {
		err = r.ConnectToNSQD(addr)
	}
}
```


## 参考资料

- [1. producer 发布msg](#1-producer-%E5%8F%91%E5%B8%83msg)
	- [1.1 msg 写入chan](#11-msg-%E5%86%99%E5%85%A5chan)
	- [1.2 连接nsqd 发布msg](#12-%E8%BF%9E%E6%8E%A5nsqd-%E5%8F%91%E5%B8%83msg)
- [2. nsqd 投递msg](#2-nsqd-%E6%8A%95%E9%80%92msg)
	- [2.1 读取msg](#21-%E8%AF%BB%E5%8F%96msg)
	- [2.2 msg 转推给consumer](#22-msg-%E8%BD%AC%E6%8E%A8%E7%BB%99consumer)
- [3. consumer 消费](#3-consumer-%E6%B6%88%E8%B4%B9)
	- [3.1 接收msg 进行处理逻辑](#31-%E6%8E%A5%E6%94%B6msg-%E8%BF%9B%E8%A1%8C%E5%A4%84%E7%90%86%E9%80%BB%E8%BE%91)
	- [3.2 与nsqd　建立连接](#32-%E4%B8%8Ensqd-%E5%BB%BA%E7%AB%8B%E8%BF%9E%E6%8E%A5)
	- [3.３ 订阅topic](#3%EF%BC%93-%E8%AE%A2%E9%98%85topic)
	- [3.４ 通知RDY](#3%EF%BC%94-%E9%80%9A%E7%9F%A5rdy)
	- [3.5 通过lookup 获取nsqds](#35-%E9%80%9A%E8%BF%87lookup-%E8%8E%B7%E5%8F%96nsqds)
- [参考资料](#%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99)