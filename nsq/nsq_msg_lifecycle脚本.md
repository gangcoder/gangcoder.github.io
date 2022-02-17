标题: nsq 源码解析:从消息发布到消费的代码实现

关键词: nsq源码,源代码阅读,golang,分布式架构,消息队列,消息系统,编程,后端服务

指导：有什么，是什么样的

摘要:

> 通过一个消息从发布到消费的代码处理过程，阅读分布式消息系统nsq 的源代码

你好，我是好刚，这一讲我们来学习nsq 的源代码。

在nsq 中，生产者发布一个消息到nsqd，nsqd 暂存消息，消费者通过nsqlookupd 查询到订阅channel 的nsqd 地址，并且连接到nsqd 上，最后nsqd 将消息转推给消费者。

整个过程可以分为4 大部分:

1. producer 发布msg
2. nsqd 转推msg
3. lookupd 管理nsqd 信息提供发现服务
4. consumer 消费msg

## 1. producer 发布msg

首先是生产者发布消息，其具体实现又可以细分为以下逻辑:

1. 获取msg 并且写入go-chan
2. 连接nsqd，取出go-chan 中的msg 发布到nsqd

### 1.1 msg 写入go-chan

获取需要发布的消息，并且将消息写入nsqd，代码实现上分为以下步骤:

1. 创建producer
2. 读取待发布的消息数据
3. 调用生产者消息发布函数
4. 创建发布消息的cmd 结构数据
5. 异步将消息写入go-chan，但是会阻塞等待消息发送并且nsqd 处理完成的响应
6. 将待发布的消息写go-入chan，由另一个goroutine 异步写入nsqd

```go
//1. 创建producer
producer, err := nsq.NewProducer(addr, cfg)

//2. 读取数据
line, readErr := r.ReadBytes(delim)

//3. 调用投递消息函数
err := producer.Publish(*topic, line)

//Publish 发布msg 到topic
func (w *Producer) Publish(topic string, body []byte) error {
	return w.sendCommand(Publish(topic, body))
}

//4. 创建发布消息的cmd，创建一条 PUB cmd
func Publish(topic string, body []byte) *Command {
	var params = [][]byte{[]byte(topic)}
	return &Command{[]byte("PUB"), params, body}
}

//5. 同步发布消息
func (w *Producer) sendCommand(cmd *Command) error {
	doneChan := make(chan *ProducerTransaction)
	err := w.sendCommandAsync(cmd, doneChan, nil) //异步发送消息

	//阻塞等待消息发送完成后的响应
	t := <-doneChan
	return t.Error
}

//6. 将待发布的消息写入chan
func (w *Producer) sendCommandAsync(cmd *Command, doneChan chan *ProducerTransaction, args []interface{})
	//连接nsqd 并且异步处理数据投递逻辑
	if atomic.LoadInt32(&w.state) != StateConnected {
		err := w.connect()
	}

	t := &ProducerTransaction{
		cmd:      cmd,
		doneChan: doneChan,
		Args:     args,
	}

	//由goroutine 异步写入nsqd
	w.transactionChan <- t:
}
```

### 1.2 连接nsqd 发布msg

在调用生产者发布消息函数的时候，生产者会建立与nsqd 的连接。

1. 创建与nsqd 的tcp 连接实例
2. 执行实例的连接函数
3. 连接完成后，开启goroutine 进行消息发送处理

```go
//建立与nsqd 的连接
func (w *Producer) connect() error {
	//1. 建立与nsqd 的连接，conn 是生产者到nsqd 的tcp 连接实例
	w.conn = NewConn(w.addr, &w.config, &producerConnDelegate{w})
	//2. 执行实例的连接操作
	_, err := w.conn.Connect()

    //3. 开启goroutine 进行消息逻辑处理
	go w.router()
}
```

处理连接，这里客户端与nsqd 的连接处理代码是生产者和消费者共用的。生产者和消费者都用这块连接的代码函数，所以函数里面有些相互兼容的写法，第一次阅读时可能会感觉奇怪，函数里面有些代码生产者不会用到，显得冗余，那这块代码可能只是给消费者用的，等所有代码看完后，就能完全理解。

```go
// NewConn 新建一个 Conn 实例
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

消息发送处理：从go-chan 中取出消息，再将消息写入到生产者与nsqd 的连接上；另外还能处理消息投递的响应结果。

```go
//goroutine 进行消息逻辑处理
func (w *Producer) router() {
	for {
		select {
		case t := <-w.transactionChan: //消息投递
			w.transactions = append(w.transactions, t)
			err := w.conn.WriteCommand(t.cmd) //写入nsqd
		case data := <-w.responseChan: //消息投递的响应结果
			w.popTransaction(FrameTypeResponse, data)
        }
    }
}

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

## 2. nsqd 转推msg

在生产者将消息发布到nsqd 后，nsqd 会先将topic 消息复制给channel 然后转推msg 到消费者，包含实现步骤分为：

1. 启动nsqd 程序，开启监听
2. 接收生产者投递到某个topic 的消息，并将消息复制到topic 的每个channel 中
3. 将topic 和channel 的注册信息通知给nsqlookupd
4. 监听消费者请求，将channel 里面的消息推送给每个订阅channel 的消费者

### 2.1 启动nsqd

nsqd 程序的入口在 `nsq/apps/nsqd/nsqd.go` 文件中，启动后执行nsqd 程序，并且开启对tcp 和http 的请求监听。

启动nsqd 的代码实现分为3 部分:

1. 获取和合并配置
2. 创建nsqd 实例
3. 执行nsqd 主逻辑：开启http, tcp 监听；启动goroutine 使用loop 持续处理各类内部数据

```go
//启动 nsqd
func (p *program) Start() error {
	//配置
    opts := nsqd.NewOptions()

    //控制台配置
	flagSet := nsqdFlagSet(opts)
	flagSet.Parse(os.Args[1:])

    //配置文件配置
	var cfg config
	configFile := flagSet.Lookup("config").Value.String()
	if configFile != "" {
		_, err := toml.DecodeFile(configFile, &cfg)
	}

    //合并解析配置
    options.Resolve(opts, flagSet, cfg)
    
    //创建nsqd 实例
	nsqd := nsqd.New(opts)

	//执行nsqd 逻辑
	nsqd.Main()

	p.nsqd = nsqd
	return nil
}

//创建NSQD 实例
//nsq/nsqd/nsqd.go
func New(opts *Options) *NSQD {
	n := &NSQD{
		startTime:            time.Now(),
		topicMap:             make(map[string]*Topic),
		exitChan:             make(chan int),
		notifyChan:           make(chan interface{}),
		optsNotificationChan: make(chan struct{}, 1),
		dl:                   dirlock.New(dataPath),
	}

	//初始化client，用于从nsqlookupd 获取集群信息
	httpcli := http_api.NewClient(nil, opts.HTTPClientConnectTimeout, opts.HTTPClientRequestTimeout)
	n.ci = clusterinfo.New(n.logf, httpcli)

	n.swapOpts(opts)
	n.errValue.Store(errStore{})

	return n
}

//Main 主逻辑
//1. 开启http, tcp 监听
//2. 开启loop 处理
func (n *NSQD) Main() {
	var err error
	ctx := &context{n}

    //端口监听
	n.tcpListener, err = net.Listen("tcp", n.getOpts().TCPAddress)
	n.httpListener, err = net.Listen("tcp", n.getOpts().HTTPAddress)
    n.httpsListener, err = tls.Listen("tcp", n.getOpts().HTTPSAddress, n.tlsConfig)

    //注册tcp 监听和nsqd 处理函数
	tcpServer := &tcpServer{ctx: ctx}
	n.waitGroup.Wrap(func() {
		protocol.TCPServer(n.tcpListener, tcpServer, n.logf)
	})
	
	//注册http/https 监听和处理handler
	httpServer := newHTTPServer(ctx, false, n.getOpts().TLSRequired == TLSRequired)
	n.waitGroup.Wrap(func() {
		http_api.Serve(n.httpListener, httpServer, "HTTP", n.logf)
	})
    httpsServer := newHTTPServer(ctx, true, true)
    n.waitGroup.Wrap(func() {
        http_api.Serve(n.httpsListener, httpsServer, "HTTPS", n.logf)
    })

    //启动goroutine 使用loop 持续处理各类内部数据
	n.waitGroup.Wrap(n.queueScanLoop) //处理消息投递后的inflight 队列和延迟队列
	n.waitGroup.Wrap(n.lookupLoop) //操作nsqlookupd，将topic 和channel 的注册注销信息发送给nsqlookupd
	n.waitGroup.Wrap(n.statsdLoop) //统计topic 和channel 各指标数据
}
```


### 2.2 接收生产者投递msg

生产者建立tcp 连接后，nsqd 能从tcp 监听中接收到生产者投递到某个topic 的msg，并且将消息存储topic 的内存go-chan 中，最后topic 内会启动goroutine 将消息复制到topic 的每个channel 中，并且返回消息发布处理的响应结果。

代码实现上，可以分为以下几步：

1. 处理tcp 连接请求，循环读取tcp 连接请求数据进行msg 发布处理
2. 将消息存储到topic 的内存go-chan 中
3. topic 内会启动goroutine 将消息复制到topic 的每个channel 中
4. 返回消息发布处理的响应结果给生产者

```go
//开启监听处理
tcpServer := &tcpServer{ctx: ctx}
protocol.TCPServer(n.tcpListener, tcpServer, n.logf)

//处理连接
//protocol.TCPServer
func TCPServer(listener net.Listener, handler TCPHandler, logf lg.AppLogFunc) {
	clientConn, err := listener.Accept()
	//针对每个连接，起一个goroutine 处理
	go handler.Handle(clientConn)
}

//处理接入请求
func (p *tcpServer) Handle(clientConn net.Conn) {
	//读取TCP 数据
	prot = &protocolV2{ctx: p.ctx}
	err = prot.IOLoop(clientConn)
}

//循环读取tcp 连接请求数据，然后根据请求数据类型进行处理
func (p *protocolV2) IOLoop(conn net.Conn) error {
	clientID := atomic.AddInt64(&p.ctx.nsqd.clientIDSequence, 1)
	client := newClientV2(clientID, conn, p.ctx)

	//启动消息推送到消费者的任务，这段代码主要针对生产者
	messagePumpStartedChan := make(chan bool)
	go p.messagePump(client, messagePumpStartedChan)
	<-messagePumpStartedChan

	for {
		//读取tcp 连接接收到的生产者数据
		line, err = client.Reader.ReadSlice('\n')
		params := bytes.Split(line, separatorBytes)

		//执行处理发布消息的请求逻辑
		var response []byte
		response, err = p.Exec(client, params) //将消息存储topic 的内存go-chan 中，topic 内会启动goroutine 将消息复制到topic 的每个channel 中

		//返回消息发布处理的响应结果给生产者
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

负责处理各类请求的函数，会将各类请求分发到具体函数上，针对生产者的消息发布请求，对应的是`PUB` 指令，处理函数会将消息暂存在topic 实例的go-chan 内存中

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

nsqd 执行接收到发布消息逻辑，实现将消息暂存在topic 实例的go-chan逻辑，代码实现分为:

1. 获取topic 名称
2. 读取message 信息
3. 获取topic 实例
4. 格式化消息体的实例
5. 将消息存储到topic 中的go-chan 上

```go
func (p *protocolV2) PUB(client *clientV2, params [][]byte) ([]byte, error) {
	//获取topic 名称
	topicName := string(params[1])

	//读取message 信息
	messageBody := make([]byte, bodyLen)
	_, err = io.ReadFull(client.Reader, messageBody)

	//获取topic 实体
	topic := p.ctx.nsqd.GetTopic(topicName)

	//格式化消息体的实例
	msg := NewMessage(topic.GenerateID(), messageBody)
	//消息投入到topic
	err = topic.PutMessage(msg)
	
	return okBytes, nil
}

//将消息存储到topic 中的go-chan 上
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

处理投递信息时，需要将信息暂存到topic 中，调用 `nsqd.GetTopic` 时，如果topic 不存在，就会实例化一个topic 对象，并且启动goroutine，这个goroutine 负责将消息复制到topic 的每个channel 中。

代码实现上，分为以下3 部分：

1. 创建topic 实例
2. topic 实例开启goroutine
3. goroutine 负责将topic go-chan 中的msg 复制到channel 的go-chan 上

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

	//将topic 创建消息通知nsqlookupd
	t.ctx.nsqd.Notify(t)

	return t
}

// messagePump goroutine 将topic 的msg 写入channel
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
		b := bufferPoolGet()
		err := writeMessageToBackend(b, m, c.backend)
		bufferPoolPut(b)
	}
	return nil
}
```

### 2.3 nsqd 通知nsqlookupd

再创建topic 实例和channel 实例时，nsqd 会向go-chan notifyChan 中写入注册消息，然后由goroutine lookupLoop 将注册信息发送到nsqlookupd。

代码实现如下：

1. 创建topic 和channel 时会向go-chan notifyChan 中写入注册消息
2. 由goroutine lookupLoop 取出注册信息并且发送到nsqlookupd。

```go
//创建topic
func NewTopic(topicName string, ctx *context, deleteCallback func(*Topic)) *Topic {
	t := &Topic{
		name:              topicName,
		channelMap:        make(map[string]*Channel),
		memoryMsgChan:     make(chan *Message, ctx.nsqd.getOpts().MemQueueSize),
	}

	t.ctx.nsqd.Notify(t)

	return t
}


//创建channel
func NewChannel(topicName string, channelName string, ctx *context,	deleteCallback func(*Channel)) *Channel {
	c := &Channel{
		topicName:      topicName,
		name:           channelName,
		memoryMsgChan:  make(chan *Message, ctx.nsqd.getOpts().MemQueueSize),
	}

	c.ctx.nsqd.Notify(c)

	return c
}

//将创建数据写入notifychan
func (n *NSQD) Notify(v interface{}) {
	n.notifyChan <- v
	err := n.PersistMetadata()
}
```

由goroutine `lookupLoop` 将注册信息发送到nsqlookupd，实现如下：

1. 开启lookup 连接
2. 定期性的向lookup 发送心跳数据
3. 从go-chan 接收topic 和channel 的变化，通知给loopup
4. 处理opt 中lookup 地址更新的消息

```go
func (n *NSQD) lookupLoop() {
	var lookupPeers []*lookupPeer //新的lookup 连接
	var lookupAddrs []string //lookup 地址

	for {
        //开启lookup 连接
        for _, host := range n.getOpts().NSQLookupdTCPAddresses {
            lookupPeer := newLookupPeer(host, n.getOpts().MaxBodySize, n.logf, connectCallback(n, hostname))
            lookupPeer.Command(nil)
            lookupPeers = append(lookupPeers, lookupPeer)
            lookupAddrs = append(lookupAddrs, host)
        }
        n.lookupPeers.Store(lookupPeers)

		select {
		case <-ticker:
			//定期性的向lookup 发送心跳数据
			for _, lookupPeer := range lookupPeers {
				cmd := nsq.Ping()
				_, err := lookupPeer.Command(cmd)
			}
		case val := <-n.notifyChan: //接收topic 和channel 的变化通知给loopup
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
		case <-n.optsNotificationChan: //处理opt 中lookup 地址更新的消息
			for _, lp := range lookupPeers {
				if in(lp.addr, n.getOpts().NSQLookupdTCPAddresses) {
					continue
				}
				lp.Close()
			}
        }
	}
}
```

### 2.4 转推消息给consumer

在消费者通过tcp 连接nsqd 后，nsqd 开启tcp 连接处理goroutine，处理消息转推逻辑，具体是将channel 的go-chan 中的数据，推送给订阅channel 的消费者。注意这里是将msg 推送给订阅消息的消费者，所以消费者需要先向nsqd 发送订阅消息。

在消费者与nsqd 连接连接后，nsqd 需要转推msg 给消费者，代码逻辑分为3 部分：

1. nsqd 开启goroutine 处理消费者的连接请求
2. 启动一个goroutine ，负责将消息推送到消费者，因为消息推送代码要先初始化，所以代码放到了前面
3. 处理消费者的各种请求，对于消费者，这里主要的请求类型有SUB 和RDY，分别表示订阅channel 和调整消费者可以接受的消息数量

为了方便理解，我们先看消费者订阅topic和channel 的处理方式，这是消费者订阅时会发送 `SUB` 命令

```go
//tcp 监听消费者请求，循环读取tcp 连接请求数据，然后根据请求数据类型进行处理
func (p *protocolV2) IOLoop(conn net.Conn) error {
	clientID := atomic.AddInt64(&p.ctx.nsqd.clientIDSequence, 1)
	client := newClientV2(clientID, conn, p.ctx)

	//启动消息推送到消费者的任务，因为消息推送代码要先初始化，所以具体实现时代码调用顺序放到了前面
	messagePumpStartedChan := make(chan bool)
	go p.messagePump(client, messagePumpStartedChan)
	<-messagePumpStartedChan

	for {
		//读取tcp 连接接收到的生产者数据
		line, err = client.Reader.ReadSlice('\n')
		params := bytes.Split(line, separatorBytes)

		//处理发布消息的请求，对于消费者，主要的请求类型有SUB 和RDY
		var response []byte
		response, err = p.Exec(client, params)

		//返回请求处理的响应结果
		if response != nil {
			err = p.Send(client, frameTypeResponse, response)
		}
	}

	//结束处理
	conn.Close()
	close(client.ExitChan)
	return err
}

//处理订阅和RDY 请求
func (p *protocolV2) Exec(client *clientV2, params [][]byte) ([]byte, error) {
	if bytes.Equal(params[0], []byte("IDENTIFY")) {
		return p.IDENTIFY(client, params)
	}
	
	switch {
	case bytes.Equal(params[0], []byte("RDY")):
		return p.RDY(client, params)
	case bytes.Equal(params[0], []byte("SUB")):
		return p.SUB(client, params)
	}
}

//nsqd 处理
//nsqd 处理订阅channel 的请求cmd
func (p *protocolV2) SUB(client *clientV2, params [][]byte) ([]byte, error) {
	topicName := string(params[1])
	channelName := string(params[2])

	//检查订阅权限
	if err := p.CheckAuth(client, "SUB", topicName, channelName); err != nil {
		return nil, err
	}

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
	//更新conn channel，将订阅的channel 写入go-chann 中
	client.SubEventChan <- channel

	return okBytes, nil
}
```

nsqd 启动goroutine 将消息推送到订阅channel 的消费者，实现分为两部分：

1. 不断从channel 的go-chan 取出msg
2. 将msg 通过tcp 连接发送给consumer

```go
func (p *protocolV2) messagePump(client *clientV2, startedChan chan bool) {
	var memoryMsgChan chan *Message
	var backendMsgChan chan []byte
	var subChannel *Channel

	//获取消费者客户端连接订阅的 channel 实例
	subEventChan := client.SubEventChan
	identifyEventChan := client.IdentifyEventChan

	for {
		//获取消息
		memoryMsgChan = subChannel.memoryMsgChan //订阅的channel 内存go-chan 中的消息
		backendMsgChan = subChannel.backend.ReadChan() //订阅的channel 持久化存储中go-chan 的消息
		flusherChan = nil
		
		select {
		case <-flusherChan:
			//flush 消息
			client.writeLock.Lock()
			err = client.Flush()
			client.writeLock.Unlock()
			flushed = true
		case <-client.ReadyStateChan:
		case subChannel = <-subEventChan: //获取消费者订阅的channel
		case b := <-backendMsgChan:
			//取出持久化存储的数据发送给consumer
			msg, err := decodeMessage(b)

			subChannel.StartInFlightTimeout(msg, client.ID, msgTimeout) //将发送的消息存储 inflight 队列中，只有收到处理正常的结果时，才从队列中移除
			client.SendingMessage() //发送的消息计数
			err = p.SendMessage(client, msg)
		case msg := <-memoryMsgChan:
			//取出内存中的数据发送给consumer
			msg.Attempts++

			subChannel.StartInFlightTimeout(msg, client.ID, msgTimeout)
			client.SendingMessage() //增加发送的消息计数
			err = p.SendMessage(client, msg)
		}
	}
}
```

发送消息处理，增加了一个buffer　处理，提高处理效率，然后调用 `Send` 将消息发送给消费者

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

## 3. lookupd 发现服务

lookupd 启动后，可以接收nsqd 传来的topic 和channel 注册信息，将这些信息存储到lookupd 的map 中，然后处理消费者查询channel 所在nsqd 的请求。

逻辑可以分为3 部分：

1. 启动lookupd 实例
2. 接收nsqd 传来的注册信息，并且将信息存储到map 中
3. 处理消费者的查询请求

### 3.1 启动lookupd 实例

启动lookupd 实例的实现又可以细分为：

1. 解析配置，包括默认配置，控制台指定配置，配置文件的配置
2. 创建nsqlookupd 实例
3. 执行nsqlookupd 实例的主逻辑：这里开启了tcp 监听，用于处理nsqd 的注册和注销请求；开启http 监听，处理消费者的查询请求

```go
//开启lookupd
func (p *program) Start() error {
    //默认配置
	opts := nsqlookupd.NewOptions()
    //控制台配置
	flagSet := nsqlookupdFlagSet(opts)
    //配置文件配置
    _, err := toml.DecodeFile(configFile, &cfg)
    //合并配置
    options.Resolve(opts, flagSet, cfg)
    
    //实例化lookup
	daemon := nsqlookupd.New(opts)

    //启动lookup
	err := daemon.Main()

    p.nsqlookupd = daemon
	return nil
}

//实例化lookupd
func New(opts *Options) *NSQLookupd {
	n := &NSQLookupd{
		opts: opts, //配置
		DB:   NewRegistrationDB(), //内存处理队列
	}
	return n
}

//执行nsqlookupd 实例的主逻辑
//这里开启了tcp 监听，用于处理nsqd 的注册和注销请求
//开启http 监听，处理消费者的查询请求
func (l *NSQLookupd) Main() error {
	ctx := &Context{l}

    //tcp 监听
	tcpListener, err := net.Listen("tcp", l.opts.TCPAddress)
	tcpServer := &tcpServer{ctx: ctx}
	go protocol.TCPServer(tcpListener, tcpServer, l.logf)

    //http 监听
    httpListener, err := net.Listen("tcp", l.opts.HTTPAddress)
    httpServer := newHTTPServer(ctx)
    go http_api.Serve(httpListener, httpServer, "HTTP", l.logf)

	return nil
}
```

### 3.2 tcp 监听处理nsqd 的请求

监听tcp 连接的处理逻辑，主要逻辑包括：接收tcp 连接，每个连接开启一个goroutine 处理tcp 上的请求数据。

具体步骤包括：

1. 读取tcp 请求数据
2. 针对不同请求的类型，对请求进行分发，交由具体函数进行逻辑处理，请求类型包括：注册，注销，ping 探活和客户端消息协商
3. 处理具体某一请求的逻辑实现

```go
//tcp 处理handle 接口
func (p *tcpServer) Handle(clientConn net.Conn) {	
    prot = &LookupProtocolV1{ctx: p.ctx}
	err = prot.IOLoop(clientConn)
}

//tcp 处理逻辑实现
func (p *LookupProtocolV1) IOLoop(conn net.Conn) error {
	client := NewClientV1(conn)
	
    //读取tcp 请求数据
    reader := bufio.NewReader(client)
	for {
		line, err = reader.ReadString('\n')
		params := strings.Split(line, " ")

        //消息逻辑处理
		var response []byte
		response, err = p.Exec(client, reader, params)
		//发回响应结果
        _, err = protocol.SendResponse(client, response)
	}
	return err
}

//Exce 逻辑，针对不同请求的类型，对请求进行分发，交由具体函数进行逻辑处理，请求类型包括：注册，注销，ping 探活和客户端消息协商
func (p *LookupProtocolV1) Exec(client *ClientV1, reader *bufio.Reader, params []string) ([]byte, error) {
	//区分请求类型
    switch params[0] {
	case "PING":
		return p.PING(client, params)
	case "IDENTIFY":
		return p.IDENTIFY(client, reader, params[1:])
	case "REGISTER":
		return p.REGISTER(client, reader, params[1:])
	case "UNREGISTER":
		return p.UNREGISTER(client, reader, params[1:])
	}
	return nil, protocol.NewFatalClientErr(nil, "E_INVALID", fmt.Sprintf("invalid command %s", params[0]))
}
```

nsqlookupd 处理nsqd 发送的注册topic 或channel 请求

```go
//注册 channel 和topic
func (p *LookupProtocolV1) REGISTER(client *ClientV1, reader *bufio.Reader, params []string) ([]byte, error) {
    //获取参数信息
	topic, channel, err := getTopicChan("REGISTER", params)

    //不过有channel 信息则注册channel
	if channel != "" {
		key := Registration{"channel", topic, channel}
		p.ctx.nsqlookupd.DB.AddProducer(key, &Producer{peerInfo: client.peerInfo}) //r.registrationMap[k][p.peerInfo.id] == p
    }

    //注册topic
	key := Registration{"topic", topic, ""}
	p.ctx.nsqlookupd.DB.AddProducer(key, &Producer{peerInfo: client.peerInfo})

	return []byte("OK"), nil
}

//将注册数据存储map 数据结构中
func (r *RegistrationDB) AddProducer(k Registration, p *Producer) bool {
	_, ok := r.registrationMap[k]
	if !ok {
		r.registrationMap[k] = make(map[string]*Producer)
	}

	producers := r.registrationMap[k]
	_, found := producers[p.peerInfo.id]
	if found == false {
		producers[p.peerInfo.id] = p
	}
}
```

### 3.3 http 监听处理消费者查询请求

监听http 请求，主要是来自消费者的查询请求，查询某个topic 的channl 列表以及相关的nsqd 列表

```go
func newHTTPServer(ctx *Context) *httpServer {
	router := httprouter.New()

    //注册路由
	router.Handle("GET", "/ping", http_api.Decorate(s.pingHandler, log, http_api.PlainText))
	//查询topic 的信息，用于发现channel 位置
    router.Handle("GET", "/lookup", http_api.Decorate(s.doLookup, log, http_api.V1))
	router.Handle("POST", "/topic/create", http_api.Decorate(s.doCreateTopic, log, http_api.V1))
    router.Handle("POST", "/channel/create", http_api.Decorate(s.doCreateChannel, log, http_api.V1))
    
	return s
}

//消费者使用 apiRequestNegotiateV1 函数查询topic 的nsqd 列表
//lookupd 中相关处理逻辑是由 doLookup 函数处理
//1. 检查topic 是否注册
//2. 获取topic 的channels
//3. 获取topic 的producers
func (s *httpServer) doLookup(w http.ResponseWriter, req *http.Request, ps httprouter.Params) (interface{}, error) {
	topicName, err := reqParams.Get("topic")

	//检查topic 是否注册
	registration := s.ctx.nsqlookupd.DB.FindRegistrations("topic", topicName, "")

	//获取topic 的channels
	channels := s.ctx.nsqlookupd.DB.FindRegistrations("channel", topicName, "*").SubKeys()

	//获取topic 的producers (包含nsqd 信息)
	producers := s.ctx.nsqlookupd.DB.FindProducers("topic", topicName, "")

	//返回结果
	return map[string]interface{}{
		"channels":  channels,
		"producers": producers.PeerInfo(),
	}, nil
}

//检查topic 是否注册
func (r *RegistrationDB) FindRegistrations(category string, key string, subkey string) Registrations {
	k := Registration{category, key, subkey}
	if _, ok := r.registrationMap[k]; ok {
		return Registrations{k}
	}
}

//获取topic 的producers 信息
func (r *RegistrationDB) FindProducers(category string, key string, subkey string) Producers {
	k := Registration{category, key, subkey}
	return ProducerMap2Slice(r.registrationMap[k])
}
```


## 4. consumer 消费msg

最后是消费者消费msg，nsq 采用消息推送的模式，所以消费者需要接收来自nsqd 主动推过来的消息，整个过程可以分为4 部分：

1. 添加自定义的消息处理逻辑
2. 查询lookupd 获取nsqds 列表
3. 建立与nsqd 连接，并且发送订阅cmd 和RDY cmd 命令
4. 接收nsqd 推送的消息并且将消息推送到go-chan，然后异步执行消息处理逻辑


### 3.1 添加消息处理逻辑

首先需要启动一个consumer 实例，然后将自定义处理逻辑传到consumer 实例上，最后启动goroutine 监听go-chan 以处理nsqd 推送过来的数据

1. 启动consumer 实例
2. 添加消息处理 handler
3. 启动goroutine ，开启消息处理监听消息写入的go-chan，有消息时执行消息处理逻辑

启动一个消费者实例，包括实例化consumer，增加处理handle，建立nsqlookupd 连接

```go
//一个典型的消费者实现逻辑
//创建消费者
consumer, err := nsq.NewConsumer(topics[i], *channel, cfg)

//添加处理handler
consumer.AddHandler(&TailHandler{topicName: topics[i], totalMessages: *totalMessages})

//建立与nsqlookupd 的连接，然后与nsqd 链接
err = consumer.ConnectToNSQLookupds(lookupdHTTPAddrs)

//直接建议与nsqd 的连接，并且消费msg
err = consumer.ConnectToNSQDs(nsqdTCPAddrs)
```

添加自定义的消息处理 handler，将处理逻辑传到consumer 实例

```go
// AddHandler 添加handler
func (r *Consumer) AddHandler(handler Handler) {
	r.AddConcurrentHandlers(handler, 1)
}
```

启动goroutine ，监听消息写入的go-chan，有消息时执行消息处理逻辑

```go
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


### 3.2 查询lookupd 获取nsqds 列表

消费者查询lookupd，获取订阅的channel 都在哪些nsqd 上，这样就能发现nsqd 的位置，然后消费者连接nsqd 接收msg

代码实现上分为2 部分:

1. 通过lookupd 查询nsqd 地址
2. 获取到nsqd 地址后进行连接处理

```go
//通过lookupd 查询nsqd 地址
// ConnectToNSQLookupd 添加lookupd 地址
func (r *Consumer) ConnectToNSQLookupd(addr string) error {
	r.lookupdHTTPAddrs = append(r.lookupdHTTPAddrs, addr)
	numLookupd := len(r.lookupdHTTPAddrs)

	// 处理第一个lookup 地址时开启goroutine
	if numLookupd == 1 {
		r.queryLookupd()
	}
}

// 查询lookup，获取topic 的producer，producer 结构体中包含nsqd 信息
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

### 3.3 建立与nsqd 连接


再拿到nsqd 地址后，与nsqd 建立连接，并且发送订阅cmd，更新RDY 状态

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

    //发送订阅topic, channel 的cmd 命令
	cmd := Subscribe(r.topic, r.channel)
	err = conn.WriteCommand(cmd)
	
    //发送RDY 消息，开始接收数据
    r.maybeUpdateRDY(c)
}

//订阅topic
// Subscribe 订阅topic 的cmd
func Subscribe(topic string, channel string) *Command {
	var params = [][]byte{[]byte(topic), []byte(channel)}
	return &Command{[]byte("SUB"), params, nil}
}

//发送RDY 消息，通知nsqd 开始推送msg 到消费者
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


### 3.4 接收nsqd 推送的消息

在连接到nsqd 后，开启goroutine 监控tcp 上推送到消费者的消息，并且将消息推送到go-chan，然后异步执行消息处理逻辑

具体分为三个步骤实现：

1. 执行连接到nsqd 命令
2. 开启goroutine 分别监听处理从tcp 连接上读数据和写数据的逻辑
3. 消费者从tcp 连接上读取nsqd 推送的数据
4. 将读取到的msg 发送到代理实例上，并且写入consumer实例的go-chan incomingMessages 中
5. consumer实例的go-chan incomingMessages 会由单独的goroutine 取出处理

```go
//执行连接到nsqd 命令
// Connect 连接nsqd
func (c *Conn) Connect() (*IdentifyResponse, error) {
	conn, err := dialer.Dial("tcp", c.addr)
	c.conn = conn.(*net.TCPConn)
	c.r = conn
	c.w = conn

	//写入协议头
	_, err = c.Write(MagicV2)

	// 开启goroutine 运行read 和 write loop，监听处理从tcp 连接上读数据和写数据的逻辑
	c.wg.Add(2)
	go c.readLoop()
	go c.writeLoop()
}

//消费者从tcp 连接上读取nsqd 推送的数据
//针对每个连接，开启goroutine 不断读取tcp 上nsqd 发过来的消息
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
		case FrameTypeMessage: //nsqd 推送到消费者的msg
			msg, err := DecodeMessage(data)
			c.delegate.OnMessage(c, msg)
		case FrameTypeError: //接收返回的错误消息
			c.delegate.OnError(c, data)
		default:
			c.delegate.OnIOError(c, fmt.Errorf("unknown frame type %d", frameType))
		}
	}
}

//将读取到的msg 发送到代理实例上，并且写入consumer实例的go-chan incomingMessages 中
//d.r 就是consumer 实例
func (d *consumerConnDelegate) OnMessage(c *Conn, m *Message)         { d.r.onConnMessage(c, m) } //消费者获取到消息 

//consumer 实例处理，将读取的msg 写入go-chan incomingMessages
func (r *Consumer) onConnMessage(c *Conn, msg *Message) {
	atomic.AddInt64(&r.totalRdyCount, -1)
	atomic.AddUint64(&r.messagesReceived, 1)
	r.incomingMessages <- msg //消息存储consumer 的go-chan，然后会由goroutine handlerLoop 取出，并且调用自定义处理handler 进行处理
	r.maybeUpdateRDY(c)
}
```

以上是nsq 的源代码实现

## 参考资料

- [1. producer 发布msg](#1-producer-发布msg)
	- [1.1 msg 写入go-chan](#11-msg-写入go-chan)
	- [1.2 连接nsqd 发布msg](#12-连接nsqd-发布msg)
- [2. nsqd 转推msg](#2-nsqd-转推msg)
	- [2.1 启动nsqd](#21-启动nsqd)
	- [2.2 接收生产者投递msg](#22-接收生产者投递msg)
	- [2.3 nsqd 通知nsqlookupd](#23-nsqd-通知nsqlookupd)
	- [2.4 转推消息给consumer](#24-转推消息给consumer)
- [3. lookupd 发现服务](#3-lookupd-发现服务)
	- [3.1 启动lookupd 实例](#31-启动lookupd-实例)
	- [3.2 tcp 监听处理nsqd 的请求](#32-tcp-监听处理nsqd-的请求)
	- [3.3 http 监听处理消费者查询请求](#33-http-监听处理消费者查询请求)
- [4. consumer 消费msg](#4-consumer-消费msg)
	- [3.1 添加消息处理逻辑](#31-添加消息处理逻辑)
	- [3.2 查询lookupd 获取nsqds 列表](#32-查询lookupd-获取nsqds-列表)
	- [3.3 建立与nsqd 连接](#33-建立与nsqd-连接)
	- [3.4 接收nsqd 推送的消息](#34-接收nsqd-推送的消息)
- [参考资料](#参考资料)