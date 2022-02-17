<!-- ---
title: nsq producer
date: 2018-08-23 10:04:58
update: 2020-03-24 18:27:28
category: language, go, nsq
--- -->

# NSQ 生产者实现

实现将消息并发布到 NSQD 的功能。

1. 创建producer
   1. 发布消息
2. 发布消息逻辑处理
   1. 创建发布消息 cmd
   2. 将待发布的消息写入chan，由goroutine 异步写入nsqd
   3. 阻塞等待消息发送完成后的响应
   4. 处理消息投递和投递响应
3. 连接nsqd 并且异步处理数据投递逻辑
   1. 建立与nsqd 的连接
   2. 连接消息的读取和写入操作

![](images/go_nsq_producer.svg)

使用示例：

```go
// 可以使用自带的to_nsq 工具，从终端发送消息到nsq
// to_nsq -topic="test" -nsqd-tcp-address="127.0.0.1:4150" -rate=10

//创建producer
producer, err := nsq.NewProducer(addr, cfg)

//进行消息投递
err := producer.Publish(*topic, line)
```

## 1. 发布消息

主要数据结构：

```go
// Producer 发布消息到nsqd
//producer 实例1对1 的连接到一个nsqd
type Producer struct {
    id     int64
    addr   string //nsqd tcp 地址
    conn   producerConn //producer 到nsqd 的连接接口
    
    responseChan chan []byte //收到响应数据后，数据写入chan // w.responseChan <- data

    transactionChan chan *ProducerTransaction //消息写入chan
    transactions    []*ProducerTransaction //投递中的传输
}

//ProducerTransaction 暂存临时消息，用于响应结果时获取消息元信息
type ProducerTransaction struct {
    cmd      *Command
    doneChan chan *ProducerTransaction
}

// Command 代码一个客户端发送给NSQ 服务的命令
type Command struct {
    Name   []byte
    Params [][]byte
    Body   []byte
}

// Conn 到nsqd 的连接
type Conn struct {
    messagesInFlight int64
    rdyCount         int64

    conn    *net.TCPConn
    addr    string

    delegate ConnDelegate

    r io.Reader
    w io.Writer

    cmdChan         chan *Command
    msgResponseChan chan *msgResponse
}
```

发布消息，创建生产者后，发布消息。

```go
//创建producer
producer, err := nsq.NewProducer(addr, cfg)

//进行消息投递
err := producer.Publish(*topic, line)
```

创建生产者实例：

```go
// github.com/nsqio/go-nsq/producer.go
// 创建生产者实例
func NewProducer(addr string, config *Config) (*Producer, error) {
    // ...
    p := &Producer{
        id: atomic.AddInt64(&instCount, 1),

        addr:   addr,
        config: *config,

        transactionChan: make(chan *ProducerTransaction),
    }
    return p, nil
}
```

消息发布：

```go
// github.com/nsqio/go-nsq/producer.go
//发布消息逻辑处理
func (w *Producer) Publish(topic string, body []byte) error {
    return w.sendCommand(Publish(topic, body))
}

//创建发布消息的cmd
func Publish(topic string, body []byte) *Command {
    var params = [][]byte{[]byte(topic)}
    return &Command{[]byte("PUB"), params, body}
}

//同步发布消息
func (w *Producer) sendCommand(cmd *Command) error {
    doneChan := make(chan *ProducerTransaction)
    err := w.sendCommandAsync(cmd, doneChan, nil) //异步发送消息

    //阻塞等待消息发送完成后的响应
    t := <-doneChan
    return t.Error
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

    // 将命令写入异步处理channel
    w.transactionChan <- t:
}
```

## 2. 生产者连接NSQD

连接nsqd 并且异步处理数据投递逻辑。

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


### 2.1 建立到 NSQD 连接

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

### 2.2 cmd 写入conn

将消息发送到nsqd

```go
// WriteCommand 将cmd 写入nsqd
func (c *Conn) WriteCommand(cmd *Command) error {
    _, err := cmd.WriteTo(c)
    err = c.Flush()
}

// WriteTo cmd 写入conn
func (c *Command) WriteTo(w io.Writer) (int64, error) {
    n, err := w.Write(c.Name)

    for _, param := range c.Params {
        n, err := w.Write(byteSpace)
        total += int64(n)

        n, err = w.Write(param)
    }

    n, err = w.Write(byteNewLine)

    if c.Body != nil {
        bufs := buf[:]
        binary.BigEndian.PutUint32(bufs, uint32(len(c.Body)))
        n, err := w.Write(bufs)

        n, err = w.Write(c.Body)
    }

    return total, nil
}
```

### 2.3 发布响应处理

消息投递结果，会异步读取。

```go
func (w *Producer) popTransaction(frameType int32, data []byte) {
    t := w.transactions[0]
    w.transactions = w.transactions[1:]
    t.finish()
}

func (t *ProducerTransaction) finish() {
    if t.doneChan != nil {
        t.doneChan <- t
    }
}
```

## 3. 响应读取 readLoop

从生产者到NSQD 的连接上读取响应数据。

1. 读取数据
2. 处理心跳数据
    1. 发送心跳
3. 针对不同消息，进行不同逻辑处理
    1. 消息投递响应
    2. 接收到投递的消息
    3. 接收到错误消息

```go
func (c *Conn) readLoop() {
    delegate := &connMessageDelegate{c}
    for {
        //读取数据
        frameType, data, err := ReadUnpackedResponse(c)
        
        //处理心跳数据
        if frameType == FrameTypeResponse && bytes.Equal(data, []byte("_heartbeat_")) {
            c.delegate.OnHeartbeat(c)
            err := c.WriteCommand(Nop()) //发送心跳
            continue
        }

        //针对不同消息，进行不同逻辑处理
        switch frameType {
        case FrameTypeResponse: //接收nsqd 返回生产者投递的消息响应
            c.delegate.OnResponse(c, data)
        case FrameTypeMessage: //接收nsqd 分发到消费者的消息
            msg, err := DecodeMessage(data)
            c.delegate.OnMessage(c, msg)
        case FrameTypeError: //接收返回的错误消息
            c.delegate.OnError(c, data)
        default:
            c.delegate.OnIOError(c, fmt.Errorf("unknown frame type %d", frameType))
        }
    }
}
```

生产者代理器处理消息投递响应结果：

```go
//代理
func (d *producerConnDelegate) OnResponse(c *Conn, data []byte)       { d.w.onConnResponse(c, data) }

//处理投递响应
func (w *Producer) onConnResponse(c *Conn, data []byte) { w.responseChan <- data }

//处理响应数据
func (w *Producer) router() {
    for {
        select {
        case data := <-w.responseChan:
            w.popTransaction(FrameTypeResponse, data)
        }
    }
}
```

## 参考资料

- github.com/nsqio/go-nsq/producer.go

