<!-- ---
title: Grpc 客户端负载均衡
date: 2020-10-28 13:24:00
category: showcode, grpc
--- -->

# Grpc 客户端负载均衡

均衡器主要逻辑：

```go
// 获取balancer 均衡器
cc.maybeApplyDefaultServiceConfig(s.Addresses)

// 处理解析器返回的地址列表
uccsErr := bw.updateClientConnState(&balancer.ClientConnState{ResolverState: s, BalancerConfig: balCfg})
```

![](images/grpc_client_balancer.svg)

内置负载均衡器包括：
1. pickfirst
2. 轮询
3. 加权轮询

均衡器主要数据结构。

```go
// ccBalancerWrapper
type ccBalancerWrapper struct {
    cc         *ClientConn
    balancer   balancer.Balancer
    subConns map[*acBalancerWrapper]struct{}
}

type Builder interface {
    // Build 创建一个负载均衡器
    Build(cc ClientConn, opts BuildOptions) Balancer
    Name() string
}

type pickfirstBalancer struct {
    state connectivity.State
    cc    balancer.ClientConn
    sc    balancer.SubConn
}

type Balancer interface {
    // UpdateClientConnState  ClientConn 状态变更时调用
    UpdateClientConnState(ClientConnState) error
    // UpdateSubConnState SubConn 状态变更时调用
    UpdateSubConnState(SubConn, SubConnState)
}

type acBalancerWrapper struct {
    ac *addrConn
}

// 地址的连接
type addrConn struct {
    dopts  dialOptions
    acbw   balancer.SubConn
    transport transport.ClientTransport
    curAddr resolver.Address
    addrs   []resolver.Address
    state connectivity.State
}

type pickerWrapper struct {
    blockingCh chan struct{}
    picker     balancer.Picker
}
```

## 1. 获取balancer 均衡器

```go
cc.maybeApplyDefaultServiceConfig(s.Addresses)
```

```go
func (cc *ClientConn) maybeApplyDefaultServiceConfig(addrs []resolver.Address) {
    // ...
    cc.applyServiceConfigAndBalancer(emptyServiceConfig, addrs)
}

// 获取均衡器
func (cc *ClientConn) applyServiceConfigAndBalancer(sc *ServiceConfig, addrs []resolver.Address) {
    // 默认均衡器
    newBalancerName = PickFirstBalancerName

    // 切换均衡器
    cc.switchBalancer(newBalancerName)
}
```

### 1.1 切换均衡创建器

```go
func (cc *ClientConn) switchBalancer(name string) {
    // 获取创建器
    builder := balancer.Get(name)
    if builder == nil {
        // 使用默认创建器
        builder = newPickfirstBuilder()
    }

    cc.curBalancerName = builder.Name()
    cc.balancerWrapper = newCCBalancerWrapper(cc, builder, cc.balancerBuildOpts)
}
```

### 1.2 均衡器处理

```go
func newCCBalancerWrapper(cc *ClientConn, b balancer.Builder, bopts balancer.BuildOptions) *ccBalancerWrapper {
    ccb := &ccBalancerWrapper{
        cc:       cc,
        scBuffer: buffer.NewUnbounded(),
        done:     grpcsync.NewEvent(),
        subConns: make(map[*acBalancerWrapper]struct{}),
    }
    
    go ccb.watcher()
    ccb.balancer = b.Build(ccb, bopts)
    return ccb
}
```

```go
func (ccb *ccBalancerWrapper) watcher() {
    for {
        select {
        case t := <-ccb.scBuffer.Get():
            // 接收地址更新消息
            // 更新子连接状态
            su := t.(*scStateUpdate)
            ccb.balancer.UpdateSubConnState(su.sc, balancer.SubConnState{ConnectivityState: su.state, ConnectionError: su.err})
        }
        // ...
    }
}
```

## 2. pickfirst 均衡器实现

```go
// 注册均衡器
func init() {
    balancer.Register(newPickfirstBuilder())
}

func Register(b Builder) {
    m[strings.ToLower(b.Name())] = b
}
```

### 2.1 创建负载均衡器

```go
func (*pickfirstBuilder) Build(cc balancer.ClientConn, opt balancer.BuildOptions) balancer.Balancer {
    return &pickfirstBalancer{cc: cc}
}
```

### 2.2 负载均衡更新子连接状态

```go
func (b *pickfirstBalancer) UpdateSubConnState(sc balancer.SubConn, s balancer.SubConnState) {
    // ...
    switch s.ConnectivityState {
    case connectivity.Ready, connectivity.Idle:
        b.cc.UpdateState(balancer.State{ConnectivityState: s.ConnectivityState, Picker: &picker{result: balancer.PickResult{SubConn: sc}}})
        // ...
    }
}
```

### 2.3 更新链接状态


```go
b.cc.UpdateState(balancer.State{ConnectivityState: s.ConnectivityState, Picker: &picker{result: balancer.PickResult{SubConn: sc}}})

func (ccb *ccBalancerWrapper) UpdateState(s balancer.State) {
    // 更新连接状态
    ccb.cc.blockingpicker.updatePicker(s.Picker)
    ccb.cc.csMgr.updateState(s.ConnectivityState)
}
```

```go
func (pw *pickerWrapper) updatePicker(p balancer.Picker) {
    // ...
    pw.picker = p
}
```

```go
func (csm *connectivityStateManager) updateState(state connectivity.State) {
    // ...
    csm.state = state
}
```

### 2.4 负载均衡更新连接

```go
func (b *pickfirstBalancer) UpdateClientConnState(cs balancer.ClientConnState) error {
    // 连接为空
    if b.sc == nil {
        // 创建子连接
        b.sc, err = b.cc.NewSubConn(cs.ResolverState.Addresses, balancer.NewSubConnOptions{})
        
        // 更新连接状态
        b.state = connectivity.Idle
        b.cc.UpdateState(balancer.State{ConnectivityState: connectivity.Idle, Picker: &picker{result: balancer.PickResult{SubConn: b.sc}}})
        
        // 子连接创建连接
        b.sc.Connect()
    }
    return nil
}
```

## 3. 创建子连接

创建子连接，并且开启网络连接。

### 3.1 创建子连接

```go
b.sc, err = b.cc.NewSubConn(cs.ResolverState.Addresses, balancer.NewSubConnOptions{})

// cc ccBalancerWrapper
func (ccb *ccBalancerWrapper) NewSubConn(addrs []resolver.Address, opts balancer.NewSubConnOptions) (balancer.SubConn, error) {
    // ...
    ac, err := ccb.cc.newAddrConn(addrs, opts)
    
    // ...
    acbw := &acBalancerWrapper{ac: ac}
    ac.acbw = acbw
    
    ccb.subConns[acbw] = struct{}{}
    return acbw, nil
}
```

创建子地址连接。

```go
func (cc *ClientConn) newAddrConn(addrs []resolver.Address, opts balancer.NewSubConnOptions) (*addrConn, error) {
    ac := &addrConn{
        state:        connectivity.Idle,
        cc:           cc,
        addrs:        addrs,
    }

    // 子地址连接map
    cc.conns[ac] = struct{}{}
    return ac, nil
}
```

### 3.2 子连接创建连接

```go
b.sc.Connect()

func (acbw *acBalancerWrapper) Connect() {
    acbw.ac.connect()
}
```

```go
func (ac *addrConn) connect() error {
    // 异步连接到指定地址
    go ac.resetTransport()
    return nil
}
```

```go
func (ac *addrConn) resetTransport() {
    // 获取地址
    addrs := ac.addrs
    
    // 连接到指定地址
    newTr, addr, reconnect, err := ac.tryAllAddrs(addrs, connectDeadline)
    
    // 连接
    ac.curAddr = addr
    ac.transport = newTr
    
    // 健康检查
    ac.startHealthCheck(hctx)
    
    // ...
}
```

尝试连接到地址。

```go
func (ac *addrConn) tryAllAddrs(addrs []resolver.Address, connectDeadline time.Time) (transport.ClientTransport, resolver.Address, *grpcsync.Event, error) {
    // ...
        
    newTr, reconnect, err := ac.createTransport(addr, copts, connectDeadline)
    return newTr, addr, reconnect, nil
}

func (ac *addrConn) createTransport(addr resolver.Address, copts transport.ConnectOptions, connectDeadline time.Time) (transport.ClientTransport, *grpcsync.Event, error) {
    // ...
    newTr, err := transport.NewClientTransport(connectCtx, ac.cc.ctx, addr, copts, onPrefaceReceipt, onGoAway, onClose)
    
    // ...
    return newTr, reconnect, nil
}
```

## 参考资料

- github.com/grpc/grpc-go/balancer_conn_wrappers.go

