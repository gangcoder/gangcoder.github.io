<!-- ---
title: Consul agent client 模式
date: 2020-06-19 15:40:57
category: showcode, consul
--- -->

# Consul agent client 模式

启动 agent 客户端模式，所有到客户端的请求，都会转发给agent 服务端处理。

可以处理：

1. 服务发现
2. 健康检查
3. 多数据中心转发

![](images/consul_agent_client.svg)

主要逻辑代码：

```go
client, err := consul.NewClientWithOptions(consulCfg, options...)
```

主要代码结构：

```go
// Client 
type Client struct {
    // Connection 到服务端的连接池
    connPool *pool.ConnPool
    // routers 请求服务路由
    routers *router.Manager
}

// 管理consul server 节点
type Manager struct {
    // consul server 节点列表
    listValue atomic.Value
}
```

## 1. 启动Agent 客户端模式

启动客户端模式。

```go
// github.com/hashicorp/consul/agent/consul/client.go
// 开启和运行consul agent 客户端模式命令
func NewClientWithOptions(config *Config, options ...ConsulOption) (*Client, error) {
    // ...
    // agent 到其他服务节点的连接池
    connPool = &pool.ConnPool{
        Server:          false,
        SrcAddr:         config.RPCSrcAddr,
        Datacenter:      config.Datacenter,
    }

    // 创建Client
    c := &Client{
        config:          config,
        connPool:        connPool,
        eventCh:         make(chan serf.Event, serfEventBacklog),
        logger:          logger.NamedIntercept(logging.ConsulClient),
    }

    // 限流器
    c.rpcLimiter.Store(rate.NewLimiter(config.RPCRate, config.RPCMaxBurst))

    // 初始化局域网Serf
    c.serf, err = c.setupSerf(config.SerfLANConfig, c.eventCh, serfLANSnapshot)

    // 创建到服务端节点的路由器，这里router 是为了选出服务端节点
    c.routers = router.New(c.logger, c.shutdownCh, c.serf, c.connPool, "")
    go c.routers.Start()

    // 开启本地局域网event 处理器
    go c.lanEventHandler()

    return c, nil
}
```

## 2. 服务请求路由

consul agent 上的请求都会代理到consul server 进行处理。服务请求路由管理consul server 节点。

创建consul server 节点路由。

```go
c.routers = router.New(c.logger, c.shutdownCh, c.serf, c.connPool, "")

func New(logger hclog.Logger, shutdownCh chan struct{}, clusterInfo ManagerSerfCluster, connPoolPinger Pinger, serverName string) (m *Manager) {
    // 创建路由实例
    m = new(Manager)

    return m
}
```

运行server 节点路由。

```go
go c.routers.Start()

// 对节点进行负载均衡处理
func (m *Manager) Start() {
    for {
        select {
        case <-m.rebalanceTimer.C:
            m.RebalanceServers()
            // ...
        }
    }
}

// 节点再均衡处理
func (m *Manager) RebalanceServers() {
    // 获取当前节点列表
    l := m.getServerList()

    // 打乱节点次序
    l.shuffleServers()
}
```

## 3. 客户端 RPC 接口实现

客户端RPC 接口实现。当agent 以客户端模式启动时，会接收来自http 的请求，客户端会将请求通过RPC 转发给agent 服务端进行处理。

```go
// 客户端模式RPC 用来向服务端发起请求
func (c *Client) RPC(method string, args interface{}, reply interface{}) error {
    // 选取服务端节点
    server := c.routers.FindServer()

    // 发起请求
    rpcErr := c.connPool.RPC(c.config.Datacenter, server.ShortName, server.Addr, method, args, reply)

    return rpcErr
}
```

```go
// 选择服务节点
func (m *Manager) FindServer() *metadata.Server {
    l := m.getServerList()
    // ...
    return l.servers[0]
}
```

```go
// RPC 请求
func (p *ConnPool) RPC(
    dc string,
    nodeName string,
    addr net.Addr,
    method string,
    args interface{},
    reply interface{},
) error {
    // ...
    return p.rpc(dc, nodeName, addr, method, args, reply)
}

func (p *ConnPool) rpc(dc string, nodeName string, addr net.Addr, method string, args interface{}, reply interface{}) error {
    p.once.Do(p.init)

    // 远程连接客户端
    conn, sc, err := p.getClient(dc, nodeName, addr)
    
    // 远程调用
    err = msgpackrpc.CallWithCodec(sc.codec, method, args, reply)
    
    // ...
    return nil
}
```

## 4. Serf 通信处理

consul agent 通过serf 获取server 节点情况。

### 4.1 开启Serf 服务

```go
// 初始化局域网Serf
c.serf, err = c.setupSerf(config.SerfLANConfig, c.eventCh, serfLANSnapshot)

// setupSerf
func (c *Client) setupSerf(conf *serf.Config, ch chan serf.Event, path string) (*serf.Serf, error) {
    conf.Init()

    conf.NodeName = c.config.NodeName
    conf.EventCh = ch

    // 创建serf 服务
    return serf.Create(conf)
}
```

### 4.2 处理Serf 消息

处理通过Serf 服务传递的事件消息。

```go
// 开启本地局域网event 处理器
go c.lanEventHandler()

// lanEventHandler is used to handle events from the lan Serf cluster
func (c *Client) lanEventHandler() {
    for {
        // ...
        select {
        case e := <-c.eventCh:
            switch e.EventType() {
            case serf.EventMemberJoin:
                // 成员加入
                c.nodeJoin(e.(serf.MemberEvent))
            case serf.EventMemberLeave, serf.EventMemberFailed, serf.EventMemberReap:
                // 成员离开
                c.nodeFail(e.(serf.MemberEvent))
        }
    }
}
```

成员加入处理：

```go
func (c *Client) nodeJoin(me serf.MemberEvent) {
    for _, m := range me.Members {
        // ...
        // 加入节点信息添加到server 服务路由器上
        c.routers.AddServer(parts)
    }
}

// 添加节点
func (m *Manager) AddServer(s *metadata.Server) {
    // 取出所有节点
    l := m.getServerList()

    // 添加新节点
    newServers := make([]*metadata.Server, len(l.servers), len(l.servers)+1)
    copy(newServers, l.servers)
    newServers = append(newServers, s)
    l.servers = newServers

    // 保存节点信息
    m.saveServerList(l)
}
```


## 参考资料

- github.com/hashicorp/consul/agent/consul/client.go

