<!-- ---
title: 告警集群实现
date: 2020-03-01 14:50:12
category: showcode, prometheus, alertmanager
--- -->

# 告警集群实现

集群节点管理基于 `Gossip` 协议，使用 `github.com/hashicorp/memberlist` 库实现。

1. 创建集群
2. 消息广播，集群中广播消息
3. 加入集群
4. 集群开启，关闭阻塞运行的 channel


![](images/alertmanager_cluster.svg)


使用示例：

```go
// 创建 cluster
peer, err = cluster.Create(...)

// 集群中广播消息
c := peer.AddState("sil", silences, ...)

// 广播消息调用
notificationLog.SetBroadcast(c.Broadcast)

// 集群节点加入
err = peer.Join(...)

// 集群开启运行，关闭阻塞运行的 channel
go peer.Settle(ctx, *gossipInterval*10)
```

## 1. 创建集群

```go
// Peer is a single peer in a gossip cluster.
type Peer struct {
	mlist    *memberlist.Memberlist
	delegate *delegate

	resolvedPeers []string

	mtx    sync.RWMutex
	states map[string]State
	stopc  chan struct{}
	readyc chan struct{}

	peerLock    sync.RWMutex
	peers       map[string]peer
	failedPeers []peer

	knownPeers    []string
	advertiseAddr string
}

// 创建 cluster
// peer, err = cluster.Create(...)
// github.com/prometheus/alertmanager/cluster/cluster.go
func Create(
	...,
	bindAddr string,
	advertiseAddr string,
	knownPeers []string,
	...,
) (*Peer, error) {
	bindHost, bindPortStr, err := net.SplitHostPort(bindAddr)
    // ...
    var advertiseHost string
	var advertisePort int
	if advertiseAddr != "" {
		var advertisePortStr string
		advertiseHost, advertisePortStr, err = net.SplitHostPort(advertiseAddr)
		// ...
	}

    // 解析其他集群节点
	resolvedPeers, err := resolvePeers(context.Background(), knownPeers, advertiseAddr, &net.Resolver{}, waitIfEmpty)

	// 创建节点名称
	name, err := ulid.New(ulid.Now(), rand.New(rand.NewSource(time.Now().UnixNano())))

    // 创建节点实例
	p := &Peer{
		states:        map[string]State{},
		stopc:         make(chan struct{}),
		readyc:        make(chan struct{}),
		logger:        l,
		peers:         map[string]peer{},
		resolvedPeers: resolvedPeers,
		knownPeers:    knownPeers,
	}

    // 创建代理器
    p.delegate = newDelegate(l, reg, p, retransmit)

	// 其余集群节点先作为失效节点进行初始化
    p.setInitialFailed(resolvedPeers, fmt.Sprintf("%s:%d", advertiseHost, advertisePort))
	
    // 创建 gossip 协议网络
    ml, err := memberlist.Create(cfg)
    
    // ...
	p.mlist = ml
	return p, nil
}

// 创建集群网络处理的代理器
func newDelegate(l log.Logger, reg prometheus.Registerer, p *Peer, retransmit int) *delegate {
    bcast := &memberlist.TransmitLimitedQueue{
		NumNodes:       p.ClusterSize,
		RetransmitMult: retransmit,
    }

    // ...
    d := &delegate{
		logger:               l,
		Peer:                 p,
		bcast:                bcast,
		messagesReceived:     messagesReceived,
		messagesReceivedSize: messagesReceivedSize,
		messagesSent:         messagesSent,
		messagesSentSize:     messagesSentSize,
		messagesPruned:       messagesPruned,
		nodeAlive:            nodeAlive,
		nodePingDuration:     nodePingDuration,
	}

	go d.handleQueueDepth()

	return d
}

// 控制暂存消息的队列长度，队列不能太长
func (d *delegate) handleQueueDepth() {
	for {
		select {
		case <-d.stopc:
			return
		case <-time.After(15 * time.Minute):
			n := d.bcast.NumQueued()
			if n > maxQueueSize {
			    // ...
				d.bcast.Prune(maxQueueSize)
			}
		}
	}
}
```

## 2. 广播消息

```go
// 集群中广播消息
c := peer.AddState("sil", silences, prometheus.DefaultRegisterer)
// 广播消息调用
notificationLog.SetBroadcast(c.Broadcast)

// AddState 创建新的广播通道
func (p *Peer) AddState(key string, s State, reg prometheus.Registerer) *Channel {
	p.states[key] = s
	send := func(b []byte) {
		p.delegate.bcast.QueueBroadcast(simpleBroadcast(b))
	}
	peers := func() []*memberlist.Node {
		nodes := p.Peers()
		for i, n := range nodes {
			if n.Name == p.Self().Name {
				nodes = append(nodes[:i], nodes[i+1:]...)
				break
			}
		}
		return nodes
    }
    
    // 消息体积过大时使用
	sendOversize := func(n *memberlist.Node, b []byte) error {
		return p.mlist.SendReliable(n, b)
	}
	return NewChannel(key, send, peers, sendOversize, p.logger, p.stopc, reg)
}

// NewChannel 创建新的消息通道
func NewChannel(
	key string,
	send func([]byte),
	peers func() []*memberlist.Node,
	sendOversize func(*memberlist.Node, []byte) error,
	logger log.Logger,
	stopc chan struct{},
	reg prometheus.Registerer,
) *Channel {
    // ...
    c := &Channel{
		key:                               key,
		send:                              send,
		peers:                             peers,
		logger:                            logger,
		msgc:                              make(chan []byte, 200),
		sendOversize:                      sendOversize,
        // ...
	}

	go c.handleOverSizedMessages(stopc)

	return c
}


// handleOverSizedMessages 过长的消息通过 channel 定时发送
func (c *Channel) handleOverSizedMessages(stopc chan struct{}) {
	var wg sync.WaitGroup
	for {
		select {
		case b := <-c.msgc:
			for _, n := range c.peers() {
				wg.Add(1)
				go func(n *memberlist.Node) {
                    // ...
					c.sendOversize(n, b)
				}(n)
			}

			wg.Wait()
		case <-stopc:
			return
		}
	}
}

// Broadcast 广播消息
func (c *Channel) Broadcast(b []byte) {
	b, err := proto.Marshal(&clusterpb.Part{Key: c.key, Data: b})
    
    // 消息太长时，写入 c.msgc ，有单独goroutine 获取后发送
	if OversizedMessage(b) {
		select {
		case c.msgc <- b:
		default:
			// ...
		}
	} else {
		c.send(b)
	}
}
```

消息接收，接收来自其他节点的数据:

```go
// 创建 Memberlist 集群实例
m, err := newMemberlist(conf)

// github.com/hashicorp/memberlist/memberlist.go
// 开启集群监听
go m.streamListen()

func (m *Memberlist) streamListen() {
	// 处理远程请求
	go m.handleConn(conn)
}

// 处理远程请求
func (m *Memberlist) handleConn(conn net.Conn) {
	// 读取消息
	msgType, bufConn, dec, err := m.readStream(conn)
	// 根据消息类型不同，进行处理
	switch msgType {
	case userMsg:
		// ...
	case pushPullMsg:
		// ...
		// 处理来源集群的数据
		m.mergeRemoteState(join, remoteNodes, userState)
	case pingMsg:
		// ...
	}
}

// 合并远程数据
func (m *Memberlist) mergeRemoteState(join bool, remoteNodes []pushNodeState, userBuf []byte) error {
	// 调用代理，合并远程数据
	m.config.Delegate.MergeRemoteState(userBuf, join)
	return nil
}


// State 可以序列化传输数据和接收远程数据
type State interface {
	MarshalBinary() ([]byte, error)
	// 接收远程数据
	Merge(b []byte) error
}

// 调用具体通道，合并数据
func (d *delegate) MergeRemoteState(buf []byte, _ bool) {
	// 解析数据
	if err := proto.Unmarshal(buf, &fs); err != nil {
		level.Warn(d.logger).Log("msg", "merge remote state", "err", err)
		return
	}
	// 合并数据
	for _, p := range fs.Parts {
		s, ok := d.states[p.Key]
		s.Merge(p.Data)
		// ...
	}
}
```

## 3. 集群节点加入

```go
// 集群节点加入
err = peer.Join(
    *reconnectInterval,
    *peerReconnectTimeout,
)

// github.com/prometheus/alertmanager/cluster/cluster.go
// 加入集群网络
func (p *Peer) Join(
	reconnectInterval time.Duration,
	reconnectTimeout time.Duration) error {
    // 加入集群网络
    n, err := p.mlist.Join(p.resolvedPeers)
    
    // 定时重连失败节点
    go p.runPeriodicTask(
        reconnectInterval,
        p.reconnect,
    )
  
    // 定时移除失败节点
    go p.runPeriodicTask(
        5*time.Minute,
        func() { p.removeFailedPeers(reconnectTimeout) },
    )

    // 定时刷新链接
	go p.runPeriodicTask(
		DefaultRefreshInterval,
		p.refresh,
	)

	return err
}
```

## 4. 集群开启

```go
// 集群开启，关闭阻塞运行的 channel
go peer.Settle(ctx, *gossipInterval*10)

func (p *Peer) Settle(ctx context.Context, interval time.Duration) {
    close(p.readyc)
}
```

## 参考资料

- github.com/prometheus/alertmanager/cluster/cluster.go
- github.com/prometheus/alertmanager/cluster/channel.go

