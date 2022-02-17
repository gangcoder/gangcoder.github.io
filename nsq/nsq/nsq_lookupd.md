<!-- ---
title: nsq lookupd
date: 2018-08-22 10:23:53
update: 2020-03-30 18:27:28
category: language, go, nsq
--- -->

# Lookupd 实现

lookupd 提供类似服务发现功能，用于获取topics 信息。

1. lookupd 启动
2. http 请求处理实现
3. tcp 请求处理实现

![](images/nsq_lookupd.svg)

## 1. lookupd 启动

主要数据结构。

```go
type NSQLookupd struct {
    sync.RWMutex
    DB           *RegistrationDB
}

type RegistrationDB struct {
    sync.RWMutex
    registrationMap map[Registration]ProducerMap
}

type Registration struct {
    Category string //类型，topic 还是channel
    Key      string //topic
    SubKey   string //channel
}

type ProducerMap map[string]*Producer

type Producer struct {
    peerInfo     *PeerInfo
    tombstoned   bool
    tombstonedAt time.Time
}

type PeerInfo struct {
    lastUpdate       int64
    id               string
    RemoteAddress    string `json:"remote_address"`
    Hostname         string `json:"hostname"`
    BroadcastAddress string `json:"broadcast_address"`
}

type ClientV1 struct {
    net.Conn
    peerInfo *PeerInfo
}
```

开启lookupd 服务。

```go
//开启lookupd
func (p *program) Start() error {
    // ...
    //默认配置
    opts := nsqlookupd.NewOptions()
    //控制台配置
    flagSet := nsqlookupdFlagSet(opts)
    
    //实例化lookup
    nsqlookupd, err := nsqlookupd.New(opts)

    p.nsqlookupd = nsqlookupd
    
    //启动lookup
    go func() {
        err := p.nsqlookupd.Main()
        // ...
    }()

    return nil
}
```

创建NSQlookupd 实例，开启网络监听。

```go
func New(opts *Options) (*NSQLookupd, error) {
    // ...
    l := &NSQLookupd{
        opts: opts,
        DB:   NewRegistrationDB(),
    }

    l.tcpListener, err = net.Listen("tcp", opts.TCPAddress)

    l.httpListener, err = net.Listen("tcp", opts.HTTPAddress)
    
    return l, nil
}
```

启动网络请求处理逻辑。

```go
//开启处理逻辑
func (l *NSQLookupd) Main() error {
    ctx := &Context{l}

    //tcp 监听
    tcpServer := &tcpServer{ctx: ctx}
    go protocol.TCPServer(l.tcpListener, tcpServer, l.logf)

    //http 监听
    httpServer := newHTTPServer(ctx)
    go http_api.Serve(l.httpListener, httpServer, "HTTP", l.logf)

    return nil
}
```

## 2. http 请求处理实现

监听http 请求，并且路由处理。

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

创建topic 请求处理：

```go
func (s *httpServer) doCreateTopic(w http.ResponseWriter, req *http.Request, ps httprouter.Params) (interface{}, error) {
    reqParams, err := http_api.NewReqParams(req)
    topicName, err := reqParams.Get("topic")

    key := Registration{"topic", topicName, ""}
    s.ctx.nsqlookupd.DB.AddRegistration(key)
    return nil, nil
}

// 注册一个topic
func (r *RegistrationDB) AddRegistration(k Registration) {
    _, ok := r.registrationMap[k]
    if !ok {
        r.registrationMap[k] = make(map[string]*Producer)
    }
}
```


consumer 查询lookup，获取channel 对应nsqd 信息

1. 检查topic 是否注册
2. 获取topic 的channels
3. 获取topic 的producers

```go
func (s *httpServer) doLookup(w http.ResponseWriter, req *http.Request, ps httprouter.Params) (interface{}, error) {
    topicName, err := reqParams.Get("topic")

    //检查topic 是否注册
    registration := s.ctx.nsqlookupd.DB.FindRegistrations("topic", topicName, "")

    //获取topic 的channels
    channels := s.ctx.nsqlookupd.DB.FindRegistrations("channel", topicName, "*").SubKeys()

    //获取topic 的producers
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

func (r *RegistrationDB) FindProducers(category string, key string, subkey string) Producers {
    k := Registration{category, key, subkey}
    return ProducerMap2Slice(r.registrationMap[k])
}
```


## 3. tcp 请求处理实现

tcp 监听处理逻辑，tcp 请求主要来自NSQD。

1. 读取tcp 接收到的数据
2. 调用handler 进行处理
3. 读取请求数据
4. 请求逻辑处理，需要区分请求类型

```go
// 开启tcp 监听
func TCPServer(listener net.Listener, handler TCPHandler, logf lg.AppLogFunc) error {
    for {
        // 读取网络数据
        clientConn, err := listener.Accept()
        // 调用处理handler 进行处理
        go handler.Handle(clientConn)
    }

    return nil
}

func (p *tcpServer) Handle(clientConn net.Conn) {
    // ...
    prot = &LookupProtocolV1{ctx: p.ctx}
    err = prot.IOLoop(clientConn)
}

//处理逻辑
func (p *LookupProtocolV1) IOLoop(conn net.Conn) error {
    client := NewClientV1(conn)
    
    //读取数据
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

func NewClientV1(conn net.Conn) *ClientV1 {
    return &ClientV1{
        Conn: conn,
    }
}
```

Exce 逻辑：

```go
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

NSQD 认证处理，确定NSQD 节点信息：

```go
func (p *LookupProtocolV1) IDENTIFY(client *ClientV1, reader *bufio.Reader, params []string) ([]byte, error) {
    // 读取节点信息
    body := make([]byte, bodyLen)
    _, err = io.ReadFull(reader, body)
    
    // 解析节点信息
    peerInfo := PeerInfo{id: client.RemoteAddr().String()}
    err = json.Unmarshal(body, &peerInfo)
    peerInfo.RemoteAddress = client.RemoteAddr().String()

    client.peerInfo = &peerInfo
    
    // 保存节点信息到服务发现中
    p.ctx.nsqlookupd.DB.AddProducer(Registration{"client", "", ""}, &Producer{peerInfo: client.peerInfo})

    // 响应数据
    data := make(map[string]interface{})
    data["tcp_port"] = p.ctx.nsqlookupd.RealTCPAddr().Port
    data["http_port"] = p.ctx.nsqlookupd.RealHTTPAddr().Port
    data["version"] = version.Binary
    response, err := json.Marshal(data)
    return response, nil
}
```

注册消息处理：

```go
//注册 channel 和topic
func (p *LookupProtocolV1) REGISTER(client *ClientV1, reader *bufio.Reader, params []string) ([]byte, error) {
    //获取参数信息
    topic, channel, err := getTopicChan("REGISTER", params)

    //注册channel
    if channel != "" {
        key := Registration{"channel", topic, channel}
        p.ctx.nsqlookupd.DB.AddProducer(key, &Producer{peerInfo: client.peerInfo})
    }

    //注册topic
    key := Registration{"topic", topic, ""}
    p.ctx.nsqlookupd.DB.AddProducer(key, &Producer{peerInfo: client.peerInfo})

    return []byte("OK"), nil
}

// add a producer to a registration
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

## 参考资料

- github.com/nsqio/nsq/apps/nsqlookupd/main.go
- github.com/nsqio/nsq/nsqlookupd/nsqlookupd.go
