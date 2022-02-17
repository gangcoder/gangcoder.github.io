<!-- ---
title: nsq consumer
date: 2018-08-23 10:05:28
update: 2020-03-24 18:27:28
category: language, go, nsq
--- -->

# NSQ 消费者实现

实现消息消费，从NSQD 取出消息并处理。

1. 创建consumer 实例
2. 添加handler
3. 连接nsqd 开始消费

![](images/go_nsq_consumer.svg)

使用示例：

```go
// 可以使用nsq 自带工具，从nsqd 获取消息并且打印到终端
// nsq_tail --topic=test --lookupd-http-address=127.0.0.1:4161

//创建消费者
consumer, err := nsq.NewConsumer(topics[i], *channel, cfg)

//添加处理handler
consumer.AddHandler(&TailHandler{topicName: topics[i], totalMessages: *totalMessages})

//建立与nsqlookupd 的连接，然后与nsqd 链接
err = consumer.ConnectToNSQLookupds(lookupdHTTPAddrs)

//直接建立与nsqd 的连接，并且消费msg
err = consumer.ConnectToNSQDs(nsqdTCPAddrs)
```

## 1. 创建消费者

主要数据结构。

```go
//Handler 处理接口
type Handler interface {
    HandleMessage(message *Message) error
}

// Consumer 消费者结构
type Consumer struct {
    topic   string
    channel string
    config  Config
    incomingMessages chan *Message
    connections        map[string]*Conn
    nsqdTCPAddrs []string
}
```


### 1.1 创建消费者

创建消费者实例。

```go
// NewConsumer 创建消费特定topic 和channel 的消费者实例
func NewConsumer(topic string, channel string, config *Config) (*Consumer, error) {
    // ...
    r := &Consumer{
        id: atomic.AddInt64(&instCount, 1),

        topic:   topic,
        channel: channel,
        config:  *config,

        incomingMessages: make(chan *Message),
        connections:        make(map[string]*Conn),
    }
    r.wg.Add(1)
    go r.rdyLoop()
    return r, nil
}
```

### 1.2 AddHandler 添加消息处理器

1. 添加消息 handler
2. 开启消息 handler loop

```go
// AddHandler 添加handler
func (r *Consumer) AddHandler(handler Handler) {
    r.AddConcurrentHandlers(handler, 1)
}

// 添加handler
func (r *Consumer) AddConcurrentHandlers(handler Handler, concurrency int) {
    //开启单独goroutine 进行消息消费处理
    go r.handlerLoop(handler)
}

// 从 incomingMessages 中取出msg 进行处理
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

        //是否禁止自动响应, 自动响应msg 处理完成消息给nsqd
        if !message.IsAutoResponseDisabled() {
            message.Finish()
        }
    }
}
```

## 2. 连接Lookupd

连接lookupd 查询nsqd 变更，然后连接nsqd 获取消息。

```go
// ConnectToNSQLookupd 添加lookupd 地址
func (r *Consumer) ConnectToNSQLookupd(addr string) error {
    //添加lookupd 地址
    r.lookupdHTTPAddrs = append(r.lookupdHTTPAddrs, addr)
    numLookupd := len(r.lookupdHTTPAddrs)

    //查询nsqd 地址
    if numLookupd == 1 {
        r.queryLookupd()
    }
}

// 查询lookup，获取topic 对应的producer，然后解析出nsqd 地址
func (r *Consumer) queryLookupd() {
    //查询lookupd
    endpoint := r.nextLookupdEndpoint()
    err := apiRequestNegotiateV1("GET", endpoint, nil, &data)
    
    //解析nsqd 地址数据
    for _, producer := range data.Producers {
        //通过producer 信息，解析出nsqd 地址
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


## 3. 连接NSQD

连接nsqd 处理实现。

```go
//连接到nsqd
func (r *Consumer) ConnectToNSQD(addr string) error {
    //创建连接实例
    conn := NewConn(addr, &r.config, &consumerConnDelegate{r})
    
    //记录正在连接中的conn
    _, pendingOk := r.pendingConnections[addr]    
    r.nsqdTCPAddrs = append(r.nsqdTCPAddrs, addr)
    
    //连接到nsqd 并且进行认证
    resp, err := conn.Connect()
    
    //订阅topic, channel
    cmd := Subscribe(r.topic, r.channel)
    err = conn.WriteCommand(cmd)
    
    //consumer 订阅完成，将conn 移入已连接map
    delete(r.pendingConnections, addr)
    r.connections[addr] = conn

    //发送RDY 消息，通知nsqd 消费者可以开始接收数据
    r.maybeUpdateRDY(c)
}
```


### 3.1 连接处理

创建连接。

```go
// NewConn 新建一个 Conn instance
func NewConn(addr string, config *Config, delegate ConnDelegate) *Conn {
    return &Conn{
        addr: addr,

        config:   config,
        delegate: delegate,

        maxRdyCount:      2500,
        lastMsgTimestamp: time.Now().UnixNano(),

        cmdChan:         make(chan *Command),
        msgResponseChan: make(chan *msgResponse),
    }
}

// Connect 连接nsqd
func (c *Conn) Connect() (*IdentifyResponse, error) {
    conn, err := dialer.Dial("tcp", c.addr)
    c.conn = conn.(*net.TCPConn)
    c.r = conn
    c.w = conn

    //写入协议头
    _, err = c.Write(MagicV2)

    // 运行read 和 write loop
    c.wg.Add(2)
    go c.readLoop()
    go c.writeLoop()
}
```


### 3.2 消息读取处理

readLoop 从连接上读取NSQD 消息。

1. 读取数据
2. 针对不同消息，进行不同逻辑处理

```go
func (c *Conn) readLoop() {
    delegate := &connMessageDelegate{c}
    for {
        //读取数据
        frameType, data, err := ReadUnpackedResponse(c)

        //针对不同消息，进行不同逻辑处理
        switch frameType {
        case FrameTypeMessage: //接收nsqd 分发到消费者的消息
            msg, err := DecodeMessage(data)
            c.delegate.OnMessage(c, msg)
        default:
            c.delegate.OnIOError(c, fmt.Errorf("unknown frame type %d", frameType))
        }
    }
}
```

### 3.3 消息处理结果回传

消息消费结果需要回传给NSQD。

```go
//消息回传
func (c *Conn) writeLoop() {
    for {
        select {
        case resp := <-c.msgResponseChan:
            c.delegate.OnMessageFinished(c, resp.msg) //c.delegate == consumerConnDelegate
            c.delegate.OnResume(c)

            //发送FIN cmd
            err := c.WriteCommand(resp.cmd)
        }
    }
}
```

### 3.4 发送订阅消息

连接到NSQD 后发送订阅topic 消息。

```go
// Subscribe 订阅topic 的cmd
func Subscribe(topic string, channel string) *Command {
    var params = [][]byte{[]byte(topic), []byte(channel)}
    return &Command{[]byte("SUB"), params, nil}
}
```

在发送RDY 消息，通知NSQD 消费者可以开始消费了。

```go
func (r *Consumer) maybeUpdateRDY(conn *Conn) {
    //动态调整RDY
    r.updateRDY(conn, count)
}

func (r *Consumer) updateRDY(c *Conn, count int64) error {
    r.sendRDY(c, count)
}


func (r *Consumer) sendRDY(c *Conn, count int64) error {
    err := c.WriteCommand(Ready(int(count)))
}
```

## 4. NSQD 连接消息代理

处理从NSQD 连接中接收消息和写入消息逻辑。

```go
// consumer 代理
type consumerConnDelegate struct {
    r *Consumer
}

// 消息处理
func (d *consumerConnDelegate) OnMessage(c *Conn, m *Message)         { d.r.onConnMessage(c, m) } 
 //consumer.onConnMessageFinished
func (d *consumerConnDelegate) OnMessageFinished(c *Conn, m *Message) { d.r.onConnMessageFinished(c, m) }
```

从conn 中读取消息，写入 incomingMessages，然后交由 Handler 处理。

```go
func (r *Consumer) onConnMessage(c *Conn, msg *Message) {
    atomic.AddInt64(&r.totalRdyCount, -1)
    atomic.AddUint64(&r.messagesReceived, 1)
    r.incomingMessages <- msg //消息存储consumer 的go-chan
    r.maybeUpdateRDY(c)
}
```

消息处理完成后发送处理结果。

```go
// Finish 消息处理完成，响应FIN 给nsqd
func (m *Message) Finish() {
    m.Delegate.OnFinish(m) // m.Delegate = connMessageDelegate
}

type connMessageDelegate struct {
    c *Conn
}

//处理finish
func (d *connMessageDelegate) OnFinish(m *Message) { d.c.onMessageFinish(m) }

//处理消息finish 数据
func (c *Conn) onMessageFinish(m *Message) {
    c.msgResponseChan <- &msgResponse{msg: m, cmd: Finish(m.ID), success: true}
}
```

## 参考资料

- github.com/nsqio/go-nsq/consumer.go

