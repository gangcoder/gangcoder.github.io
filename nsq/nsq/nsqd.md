<!-- ---
title: nsq nsqd
date: 2018-08-06 19:14:27
category: language, go, nsq, nsqd
--- -->

# NSQD 服务实现

1. 启动nsqd
   1. 启动 nsqd
   2. 创建NSQD 实例
   3. Main 主逻辑
2. http 请求处理实现
3. tcp 请求处理实现
4. messagePump 消息分发实现

![](images/nsqd.svg)

## 1. 启动NSQD

主要数据结构：

```go
type NSQD struct {    
    topicMap map[string]*Topic
    lookupPeers atomic.Value
    
    tcpListener   net.Listener
    httpListener  net.Listener
    httpsListener net.Listener
    
    ci *clusterinfo.ClusterInfo
}

type ClusterInfo struct {
    client *http_api.Client
}

// topic 数据结构
type Topic struct {
	name              string
	channelMap        map[string]*Channel
	backend           BackendQueue
	memoryMsgChan     chan *Message
}

// Channel 数据结构
type Channel struct {
	topicName string
	name      string
	memoryMsgChan chan *Message
	clients        map[int64]Consumer
}
```

启动 NSQD，NSQD 使用 `go-svc` 库，通过 `Start` 函数启动。

```go
//配置
opts := nsqd.NewOptions()

//创建nsqd 实例
nsqd, err := nsqd.New(opts)

//执行主程序
go func() {
    err := p.nsqd.Main()
    // ...
}()
```

创建NSQD 实例：

```go
//nsq/nsqd/nsqd.go
func New(opts *Options) *NSQD {
    n := &NSQD{
        startTime:            time.Now(),
        topicMap:             make(map[string]*Topic),
        notifyChan:           make(chan interface{}),
    }

    //初始化client，用于从nsqlookupd 获取集群信息
    httpcli := http_api.NewClient(nil, opts.HTTPClientConnectTimeout, opts.HTTPClientRequestTimeout)
    n.ci = clusterinfo.New(n.logf, httpcli)

    //tpc 协议监听
    n.tcpListener, err = net.Listen("tcp", opts.TCPAddress)
    
    //http 协议监听
    n.httpListener, err = net.Listen("tcp", opts.HTTPAddress)
    
    //https 协议监听
    n.httpsListener, err = tls.Listen("tcp", opts.HTTPSAddress, n.tlsConfig)

    return n
}
```

Main 主逻辑处理，包括：

1. 开启http, tcp 消息处理
2. 开启队列loop

```go
func (n *NSQD) Main() {
    var err error
    ctx := &context{n}

    //注册tcp 监听和nsqd 处理函数
    tcpServer := &tcpServer{ctx: ctx}
    n.waitGroup.Wrap(func() {
        exitFunc(protocol.TCPServer(n.tcpListener, tcpServer, n.logf))
    })
    
    //注册http/https 监听和处理handler
    httpServer := newHTTPServer(ctx, false, n.getOpts().TLSRequired == TLSRequired)
    n.waitGroup.Wrap(func() {
        exitFunc(http_api.Serve(n.httpListener, httpServer, "HTTP", n.logf))
    })
    
    // 信息收集任务队列处理
    n.waitGroup.Wrap(n.queueScanLoop)
    n.waitGroup.Wrap(n.lookupLoop)
    n.waitGroup.Wrap(n.statsdLoop)
}
```

## 2. http 监听实现

基于http 包开启http 监听，使用httprouter 注册路由，用于处理http 请求。

```go
func newHTTPServer(ctx *context, tlsEnabled bool, tlsRequired bool) *httpServer {
    router := httprouter.New()
    s := &httpServer{
        ctx:         ctx,
        router:      router,
    }

    //注册处理路由
    router.Handle("POST", "/pub", http_api.Decorate(s.doPUB, http_api.V1))
    router.Handle("GET", "/stats", http_api.Decorate(s.doStats, log, http_api.V1))

    return s
}
```

发布消息handle：

```go
func (s *httpServer) doPUB(w http.ResponseWriter, req *http.Request, ps httprouter.Params) (interface{}, error) {
    //从请求中获取消息内容
    body, err := ioutil.ReadAll(io.LimitReader(req.Body, readMax))
    
    //从请求中获取topic 名称，没有就创建
    reqParams, topic, err := s.getTopicFromQuery(req)
    
    //基于上传的数据，创建msg 实例
    msg := NewMessage(topic.GenerateID(), body)
    
    //将消息投送到topic 中
    err = topic.PutMessage(msg)
    
    return "OK", nil
}

// PutMessage 将消息保存到topic 结构体的channel 中
func (t *Topic) PutMessage(m *Message) error {
	// ...
	err := t.put(m)
}

func (t *Topic) put(m *Message) error {
	select {
	case t.memoryMsgChan <- m:
	// ...
	}
	return nil
}
```

获取统计信息：

```go
func (s *httpServer) doStats(w http.ResponseWriter, req *http.Request, ps httprouter.Params) (interface{}, error) {
    //获取参数
    topicName, _ := reqParams.Get("topic")
    channelName, _ := reqParams.Get("channel")
    
    //获取nsqd 统计信息
    stats := s.ctx.nsqd.GetStats(topicName, channelName, includeClients)
    health := s.ctx.nsqd.GetHealth()
    startTime := s.ctx.nsqd.GetStartTime()
    uptime := time.Since(startTime)

    //获取内存信息
    ms := getMemStats()
    
    //返回数据
    return struct {
        Version   string        `json:"version"`
        Health    string        `json:"health"`
        StartTime int64         `json:"start_time"`
        Topics    []TopicStats  `json:"topics"`
        Memory    memStats      `json:"memory"`
        Producers []ClientStats `json:"producers"`
    }{version.Binary, health, startTime.Unix(), stats, ms, producerStats}, nil
}
```

## 3. tcp 监听

监听tcp 连接，并且处理tcp 连接上的逻辑。

```go
// github.com/nsqio/nsq/nsqd/nsqd.go
//开启监听处理
tcpServer := &tcpServer{ctx: ctx}
protocol.TCPServer(n.tcpListener, tcpServer, n.logf)

//处理连接
func TCPServer(listener net.Listener, handler TCPHandler, logf lg.AppLogFunc) error {
    clientConn, err := listener.Accept()
    go handler.Handle(clientConn)
}

//处理接入请求
func (p *tcpServer) Handle(clientConn net.Conn) {
    //读取TCP 数据
    prot = &protocolV2{ctx: p.ctx}
    err = prot.IOLoop(clientConn)
}
```

TCP 连接处理 IOLoop，循环读取tcp 连接请求数据进行处理：

1. 启动消息推送到consumer 的任务
2. 读取tcp 连接接收到的数据
3. 根据读取到的数据，执行处理逻辑
4. 返回处理结果响应

```go
//循环读取tcp 连接请求数据，然后处理
func (p *protocolV2) IOLoop(conn net.Conn) error {
    client := newClientV2(clientID, conn, p.ctx)

    //启动消息推送到consumer 的任务，只用于消费者
    messagePumpStartedChan := make(chan bool)
    go p.messagePump(client, messagePumpStartedChan)
    <-messagePumpStartedChan

    //用于消费者和生产者
    for {
        //读取tcp 连接接收到的数据
        line, err = client.Reader.ReadSlice('\n')
        params := bytes.Split(line, separatorBytes)

        //根据读取到的数据，执行处理逻辑
        var response []byte
        response, err = p.Exec(client, params)
    }
    // ...
    return err
}
```

处理具体请求，将各类请求分发到具体函数上。

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

发布消息：

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

订阅topic：

```go
//订阅topic
func (p *protocolV2) SUB(client *clientV2, params [][]byte) ([]byte, error) {
    topicName := string(params[1])
    channelName := string(params[2])
    
    //更新topic 的channel 列表
    var channel *Channel
    for {
        topic := p.ctx.nsqd.GetTopic(topicName)
        channel = topic.GetChannel(channelName)
        channel.AddClient(client.ID, client)
        break
    }

    //更新conn 的channel 信息
    atomic.StoreInt32(&client.State, stateSubscribed)
    client.Channel = channel
    //更新conn channel
    client.SubEventChan <- channel

    return okBytes, nil
}
```

messagePump 消息发送处理：

messagePump 将消息发送给消费者客户端。

```go
// go p.messagePump(client, messagePumpStartedChan)
// github.com/nsqio/nsq/nsqd/protocol_v2.go
func (p *protocolV2) messagePump(client *clientV2, startedChan chan bool) {
    var memoryMsgChan chan *Message
    var backendMsgChan chan []byte
    var subChannel *Channel

    //获取client 连接的消费者所定于的 channel
    subEventChan := client.SubEventChan

    for {
        //获取消息
        memoryMsgChan = subChannel.memoryMsgChan
        backendMsgChan = subChannel.backend.ReadChan()
        
        select {
        case subChannel = <-subEventChan: //获取消费者订阅的channel
        case b := <-backendMsgChan:
            //发送持久存储的数据给consumer
            msg, err := decodeMessage(b)

            err = p.SendMessage(client, msg)
        case msg := <-memoryMsgChan:
            //发送内存中的数据给consumer
            msg.Attempts++

            // ...
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
    //将响应写回
    _, err := protocol.SendFramedResponse(client.Writer, frameType, data)
    
    return err
}

// SendFramedResponse 发送响应到客户端
func SendFramedResponse(w io.Writer, frameType int32, data []byte) (int, error) {
    // ...
    n, err = w.Write(data)
    return n + 8, err
}
```


## 4. Topic 消息分发实现

初始化topic 处理逻辑。

1. 初始化topic
2. 初始化topic msgpump，将topic 的消息，分发给各个channel

```go
// GetTopic 执行topic 创建操作
func (n *NSQD) GetTopic(topicName string) *Topic {
    //新建topic
    t = NewTopic(topicName, &context{n}, deleteCallback)
    n.topicMap[topicName] = t

    //连接lookup
    lookupdHTTPAddrs := n.lookupdHTTPAddrs()
    if len(lookupdHTTPAddrs) > 0 {
        channelNames, err := n.ci.GetLookupdTopicChannels(t.name, lookupdHTTPAddrs)
        
        //更新channel
        for _, channelName := range channelNames {
            t.GetChannel(channelName)
        }
    }
    
    // 开启 topic messagePump
    t.Start()
    return t
}

//topic 消息驱动
//创建一个topic 实例，并且开启goroutine 处理从topic 到channel 的消息
func NewTopic(topicName string, ctx *context, deleteCallback func(*Topic)) *Topic {
    //创建一个topic 实例
    t := &Topic{
        name:              topicName,
        channelMap:        make(map[string]*Channel),
        memoryMsgChan:     make(chan *Message, ctx.nsqd.getOpts().MemQueueSize),
    }

    //开启goroutine 处理从topic 到channel 的消息
    t.waitGroup.Wrap(t.messagePump)
    // ...
    return t
}
```

创建消费channel：

```go
func (t *Topic) GetChannel(channelName string) *Channel {
	channel, isNew := t.getOrCreateChannel(channelName)
	// ...
	return channel
}

// this expects the caller to handle locking
func (t *Topic) getOrCreateChannel(channelName string) (*Channel, bool) {
	channel = NewChannel(t.name, channelName, t.ctx, deleteCallback)
	t.channelMap[channelName] = channel

	return channel, false
}

func NewChannel(topicName string, channelName string,...) *Channel {
	c := &Channel{
		topicName:      topicName,
		name:           channelName,
		memoryMsgChan:  make(...),
		clients:        make(map[int64]Consumer),
	}
	// ...
	return c
}
```

topic 消息分发处理：

```go
// messagePump 将topic 的msg 写入channel
func (t *Topic) messagePump() {
    var msg *Message
    var chans []*Channel
    var memoryMsgChan chan *Message
    var backendChan chan []byte

    //获取topic 的channel 列表
    for _, c := range t.channelMap {
        chans = append(chans, c)
    }
    
    //topic 的内存消息go-chan 和持久化消息go-chan
    if len(chans) > 0 && !t.IsPaused() {
        memoryMsgChan = t.memoryMsgChan
        backendChan = t.backend.ReadChan()
    }

    // main message loop
    for {
        select {
        case msg = <-memoryMsgChan:
        case buf = <-backendChan:
            msg, err = decodeMessage(buf)
        }

        //msg 复制到每个channel
        for i, channel := range chans {
            chanMsg := msg
            //往每个channel 发送msg
            err := channel.PutMessage(chanMsg)
        }
    }
}

//msg 写入channel 的chan
func (c *Channel) PutMessage(m *Message) error {
    err := c.put(m)
    return nil
}

//msg 写入channel 的chan
func (c *Channel) put(m *Message) error {
    select {
    case c.memoryMsgChan <- m:
    default:
        // ...
        err := writeMessageToBackend(b, m, c.backend)
    }
    return nil
}
```

## 5. nsqd 消息通知lookupd

将nsqd 的topic 和channel 消息同步给lookupd。

1. 开启lookup 连接
2. 接收topic 和channel 的变化通知给loopup

```go
func (n *NSQD) lookupLoop() {
    var lookupPeers []*lookupPeer //新的lookup 连接

    for {
        //开启lookup 连接
        for _, host := range n.getOpts().NSQLookupdTCPAddresses {
            lookupPeer := newLookupPeer(host, n.getOpts().MaxBodySize, n.logf, connectCallback(n, hostname))
            lookupPeers = append(lookupPeers, lookupPeer)
        }

        select {
        case val := <-n.notifyChan:
            // 接收topic 和channel 的变化通知给loopup
            switch val.(type) {
            case *Channel:
                //channel 消息：上线下线channel
                channel := val.(*Channel)
                if channel.Exiting() == true {
                    cmd = nsq.UnRegister(channel.topicName, channel.name)
                } else {
                    cmd = nsq.Register(channel.topicName, channel.name)
                }
            case *Topic:
                // topic 消息
                topic := val.(*Topic)
                if topic.Exiting() == true {
                    cmd = nsq.UnRegister(topic.name, "")
                } else {
                    cmd = nsq.Register(topic.name, "")
                }
            }

            //发送消息
            for _, lookupPeer := range lookupPeers {
                _, err := lookupPeer.Command(cmd)
            }
        }
    }
}
```


## 参考资料

- github.com/nsqio/nsq/nsqd/nsqd.go

