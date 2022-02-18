<!-- ---
title: nsq conn
date: 2018-08-30 21:04:01
category: language, go, nsq
--- -->

nsq conn

nsq 中的各种tcp 连接，主要分为生产者和消费者连接nsqd，以及nsqd连接 nsqlookupd 的


## client to nsqd

创建客户端到nsqd 的连接，consumer和producer 共用这段连接处理代码。

Conn 数据结构

```go
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


// Connect 执行连接处理，连接到nsqd
//连接后进行客户端认证和协议协商
//开启goroutine 分别处理tcp 连接上的数据读写
func (c *Conn) Connect() (*IdentifyResponse, error) {
	conn, err := dialer.Dial("tcp", c.addr)
	c.conn = conn.(*net.TCPConn)
	c.r = conn
	c.w = conn

	//写入协议头
    _, err = c.Write(MagicV2)
    
    //发送identify cmd 认证信息
	resp, err := c.identify()
	if resp != nil && resp.AuthRequired {
        //发送auth cmd 验证信息
		err := c.auth(c.config.AuthSecret)
	}

	// 运行read 和 write loop
	c.wg.Add(2)
	go c.readLoop()
	go c.writeLoop()
}
```


### 客户端identify 认证

协商客户端和服务端的通信信息

```go
func (c *Conn) identify() (*IdentifyResponse, error) {
	ci := make(map[string]interface{})
	ci["client_id"] = c.config.ClientID
	ci["hostname"] = c.config.Hostname
    
    //发送cmd
	err = c.WriteCommand(cmd)
    
    //读取响应
	frameType, data, err := ReadUnpackedResponse(c)
    
    //解析结果
	resp := &IdentifyResponse{}
	err = json.Unmarshal(data, resp)
    
    //客户端处理方式更新
	if resp.TLSv1 {
		err := c.upgradeTLS(c.config.TlsConfig)
	}
	if resp.Deflate {
		err := c.upgradeDeflate(c.config.DeflateLevel)
	}
	if resp.Snappy {
		err := c.upgradeSnappy()
	}

	c.r = bufio.NewReader(c.r)
	if _, ok := c.w.(flusher); !ok {
		c.w = bufio.NewWriter(c.w)
	}

	return resp, nil
}
```

### cmd 写入conn

所有客户端写入nsqd 和nsqd 写入客户端的请求，都会被封装成`Command` 结构体

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


### readLoop

从tcp 连接上读取请求数据进行处理，如果是消费者，则主要读取nsqd 推送过来的msg 消息；如果是生产者，就是发布消息后nsqd 返回的响应结果。

1. 从tcp 连接上读取数据
2. 对于心跳类型请求，主要逻辑是发回一个心跳cmd
3. 针对不同请求，进行不同逻辑处理
	1. 生产者发布后的响应请求
	2. 消费者接收到nsqd 的消息推送数据
	3. tcp 上错误类型的请求

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
		case FrameTypeResponse: //消息投递响应
			c.delegate.OnResponse(c, data)
		case FrameTypeMessage: //接收到投递的消息
			msg, err := DecodeMessage(data)
			c.delegate.OnMessage(c, msg)
		case FrameTypeError: //接收到错误消息
			c.delegate.OnError(c, data)
		default:
			c.delegate.OnIOError(c, fmt.Errorf("unknown frame type %d", frameType))
		}
	}
}
```


### writeLoop

客户端往tcp 连接上写入请求数据，主要是消费者使用，在处理nsqd 推送的消息后，发送是否处理完成的结果请求给nsqd。

1. 处理cmd go-chan 中传入的消息
2. 消息者消费处理结果发送

```go
func (c *Conn) writeLoop() {
	for {
		//写入loop
		select {
		case <-c.exitChan:
			//exit
			close(c.drainReady)
			goto exit
		case cmd := <-c.cmdChan:
			//获取到cmd
			err := c.WriteCommand(cmd)
		case resp := <-c.msgResponseChan:
			//消息消费结果处理
			if resp.success {
				c.delegate.OnMessageFinished(c, resp.msg)
				c.delegate.OnResume(c)
			} else {
				c.delegate.OnMessageRequeued(c, resp.msg)
				if resp.backoff {
					c.delegate.OnBackoff(c)
				} else {
					c.delegate.OnContinue(c)
				}
			}

			err := c.WriteCommand(resp.cmd)
		}
	}
}
```


## nsqd to lookup

nsqd 会与nsqlookupd 建立tcp 连接，在创建topic 或者channel 后，会往nsqdlookupd 发送注册消息。这样nsqlookupd 能维护一份nsqd 和channel 的对应表。

同时nsqlookupd 会提供http 接口，主要工作消费者查询它所订阅的channel 在那些nsqd 上。

### nsqd 注册信息到nsqlookupd

nsqd 在主函数逻辑中，会开启goroutine 处理注册信息发送到nsqlookupd 的逻辑。

代码实现上分为以下4 步：

1. 开启lookup 连接
2. 定期性的向lookup 发送心跳数据
3. 接收topic 和channel 的变化通知给loopup
4. 处理opt 中lookup 地址更新的消息

```go
func (n *NSQD) lookupLoop() {
	var lookupPeers []*lookupPeer //新的lookup 连接
	var lookupAddrs []string //lookup 地址

	for {
        //开启lookup 连接
        for _, host := range n.getOpts().NSQLookupdTCPAddresses {
            lookupPeer := newLookupPeer(host, n.getOpts().MaxBodySize, n.logf, connectCallback(n, hostname))
            lookupPeer.Command(nil) //空命令，打开连接
            lookupPeers = append(lookupPeers, lookupPeer)
            lookupAddrs = append(lookupAddrs, host)
        }
        n.lookupPeers.Store(lookupPeers)
	}

	select {
	case <-ticker:
		//定期发送心跳
		for _, lookupPeer := range lookupPeers {
			cmd := nsq.Ping()
			_, err := lookupPeer.Command(cmd)
		}
	case val := <-n.notifyChan:
		//接收topic 和channel 的变化通知给loopup
		var cmd *nsq.Command
		switch val.(type) {
		case *Channel:
			branch = "channel"
			channel := val.(*Channel)
			if channel.Exiting() == true {
				cmd = nsq.UnRegister(channel.topicName, channel.name) //注册channel
			} else {
				cmd = nsq.Register(channel.topicName, channel.name) //注销channel
			}
		case *Topic:
			branch = "topic"
			topic := val.(*Topic)
			if topic.Exiting() == true {
				cmd = nsq.UnRegister(topic.name, "")
			} else {
				cmd = nsq.Register(topic.name, "")
			}
		}

		//发送cmd
		for _, lookupPeer := range lookupPeers {
			_, err := lookupPeer.Command(cmd)
		}
	case <-n.optsNotificationChan: //处理opt 中lookup 地址更新的消息
		for _, lp := range lookupPeers {
			if in(lp.addr, n.getOpts().NSQLookupdTCPAddresses) {
				tmpPeers = append(tmpPeers, lp)
				tmpAddrs = append(tmpAddrs, lp.addr)
				continue
			}
		}
	}
}

//lookupPeer 数据结构
//lookupPeer 表示nsqd 到nsqlookupd 的底层tcp 连接
type lookupPeer struct {
	logf            lg.AppLogFunc
	addr            string
	conn            net.Conn
}

//发送命令
func (lp *lookupPeer) Command(cmd *nsq.Command) ([]byte, error) {
	initialState := lp.state
	//没有连接到nsqlookup 时启动连接
	if lp.state != stateConnected {
		err := lp.Connect()
		lp.state = stateConnected
		_, err = lp.Write(nsq.MagicV1)
	}

	//发送cmd
	_, err := cmd.WriteTo(lp)
	//读取响应
	resp, err := readResponseBounded(lp, lp.maxBodySize)
	return resp, nil
}

//Connect 连接到指定地址
func (lp *lookupPeer) Connect() error {
	conn, err := net.DialTimeout("tcp", lp.addr, time.Second)
	lp.conn = conn
	return nil
}

//发布注册 cmd
func Register(topic string, channel string) *Command {
	params := [][]byte{[]byte(topic)}
	if len(channel) > 0 {
		params = append(params, []byte(channel))
	}
	return &Command{[]byte("REGISTER"), params, nil}
}

//发布删除 cmd
func UnRegister(topic string, channel string) *Command {
	params := [][]byte{[]byte(topic)}
	if len(channel) > 0 {
		params = append(params, []byte(channel))
	}
	return &Command{[]byte("UNREGISTER"), params, nil}
}

//探活消息
func Ping() *Command {
	return &Command{[]byte("PING"), nil, nil}
}
```

### lookupd

nsqlookupd 程序中会开启http/tcp 端口监听，主要通过tcp 处理来自nsqd 的注册消息，同时主要通过http 接口提供查询功能。

代码实现上分为3 部分：

1. 启动nsqlookupd， 注册http/tcp 监听端口
2. 实现tcp 监听逻辑，接收tcp 请求并且处理
3. 实现http 监听逻辑，接收http 请求

```go
//开启处理逻辑
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

tcp 监听处理逻辑

```go
//处理tcp 逻辑
func (p *LookupProtocolV1) IOLoop(conn net.Conn) error {
	client := NewClientV1(conn)
	
    //读取数据
    reader := bufio.NewReader(client)
	for {
		line, err = reader.ReadString('\n')
		params := strings.Split(line, " ")

        //逻辑处理
		var response []byte
		response, err = p.Exec(client, reader, params)
        _, err = protocol.SendResponse(client, response)
	}
	return err
}

//tcp 请求处理，根据请求类型不同，调用不同函数处理
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


//注册 channel 和topic
func (p *LookupProtocolV1) REGISTER(client *ClientV1, reader *bufio.Reader, params []string) ([]byte, error) {
    //获取参数信息
	topic, channel, err := getTopicChan("REGISTER", params)

    //注册channel
	if channel != "" {
		key := Registration{"channel", topic, channel}
		p.ctx.nsqlookupd.DB.AddProducer(key, &Producer{peerInfo: client.peerInfo}) //r.registrationMap[k][p.peerInfo.id] == p
    }

    //注册topic
	key := Registration{"topic", topic, ""}
	p.ctx.nsqlookupd.DB.AddProducer(key, &Producer{peerInfo: client.peerInfo})

	return []byte("OK"), nil
}
```

监听http 请求，并且路由处理

```go
func newHTTPServer(ctx *Context) *httpServer {
	router := httprouter.New()

    //路由处理
	router.Handle("GET", "/ping", http_api.Decorate(s.pingHandler, log, http_api.PlainText))
    router.Handle("GET", "/info", http_api.Decorate(s.doInfo, log, http_api.V1))
	
	router.Handle("GET", "/lookup", http_api.Decorate(s.doLookup, log, http_api.V1))
	router.Handle("GET", "/topics", http_api.Decorate(s.doTopics, log, http_api.V1))
	router.Handle("GET", "/channels", http_api.Decorate(s.doChannels, log, http_api.V1))
	
	// only v1
	router.Handle("POST", "/topic/create", http_api.Decorate(s.doCreateTopic, log, http_api.V1))
    router.Handle("POST", "/channel/create", http_api.Decorate(s.doCreateChannel, log, http_api.V1))
    
	return s
}
```
