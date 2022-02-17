<!-- ---
title: 基于hashicorp/raft的分布式一致性实战教学
date: 2020-08-06 19:13:43
category: showcode, consul
--- -->

# Consul Raft 实现

Consul Raft 强一致数据同步实现。

![](images/consul_raft.svg)

启动Raft 服务：

```go
// 初始化Raft 服务
s.setupRaft()
```

主要数据结构：

```go
// Raft 结构体
type Raft struct {
    // 业务状态机
    fsm FSM
    // 稳定存储
    stable StableStore
    // 节点通信
    trans Transport
}

// 状态机
type FSM struct {
    // 消息类型及其处理handler map
    apply map[structs.MessageType]command
    state     *state.Store
}

// 批量处理状态机
type ChunkingFSM struct {
    underlying raft.FSM
    store      ChunkStorage
}
```

## 1. 创建Raft 服务

```go
// setupRaft 初始化Raft 服务
func (s *Server) setupRaft() error {
    // 创建有限状态机
    s.fsm, err = fsm.New(s.tombstoneGC, s.logger)

    // 节点间通信
    trans := raft.NewNetworkTransportWithConfig(transConfig)
    s.raftTransport = trans
    
    // 创建 raft 所需存储器
    store := raft.NewInmemStore()
    s.raftInmem = store
    stable = store
    log = store
    snap = raft.NewInmemSnapshotStore()

    // 启动服务
    if s.config.Bootstrap || s.config.DevMode {
        configuration := raft.Configuration{
            Servers: []raft.Server{
                {
                    ID:      s.config.RaftConfig.LocalID,
                    Address: trans.LocalAddr(),
                },
            },
        }
        raft.BootstrapCluster(s.config.RaftConfig, log, stable, snap, trans, configuration)
    }

    // 创建Raft 实例
    s.raft, err = raft.NewRaft(s.config.RaftConfig, s.fsm.ChunkingFSM(), log, stable, snap, trans)

    return nil
}
```

创建 FMS。

```go
// New 创建fms
func New(gc *state.TombstoneGC, logger hclog.Logger) (*FSM, error) {
    // 存储器
    stateNew, err := state.NewStateStore(gc)
    
    fsm := &FSM{
        apply:  make(map[structs.MessageType]command),
        state:  stateNew,
        gc:     gc,
    }

    // 注册业务处理handler
    for msg, fn := range commands {
        thisFn := fn
        fsm.apply[msg] = func(buf []byte, index uint64) interface{} {
            return thisFn(fsm, buf, index)
        }
    }

    fsm.chunker = raftchunking.NewChunkingFSM(fsm, nil)
    return fsm, nil
}

// github.com/hashicorp/consul/agent/consul/fsm/commands_oss.go
registerCommand(structs.KVSRequestType, (*FSM).applyKVSOperation)

// 业务消息处理handler
func registerCommand(msg structs.MessageType, fn unboundCommand) {
    // ...
    commands[msg] = fn
}

// 创建批量处理状态机
func NewChunkingFSM(underlying raft.FSM, store ChunkStorage) *ChunkingFSM {
    ret := &ChunkingFSM{
        underlying: underlying,
        store:      store,
    }
    return ret
}
```

## 2. 提交Raft 数据

提交Raft 数据，数据必须在主节点上提交。

```go
s.raft.Apply(buf, enqueueLimit)

func (r *Raft) Apply(cmd []byte, timeout time.Duration) ApplyFuture {
    return r.ApplyLog(Log{Data: cmd}, timeout)
}

func (r *Raft) ApplyLog(log Log, timeout time.Duration) ApplyFuture {
    // ...
    // 提交一个log 数据
    logFuture := &logFuture{
        log: Log{
            Type:       LogCommand,
            Data:       log.Data,
            Extensions: log.Extensions,
        },
    }
    logFuture.init()

    select {
    // ...
    case r.applyCh <- logFuture:
        // 等待log 数据异步被消费
        return logFuture
    }
}
```

## 3. 接收Raft 同步数据

```go
// follower 节点 Raft 内收到 LogCommand 类型的消息时，会调用 fsm 进行业务数据同步
resp = r.fsm.Apply(req.log)

func (c *ChunkingFSM) Apply(l *raft.Log) interface{} {
    // 如果不是批量处理，则调用外层状态机处理
    if l.Type != raft.LogCommand || l.Extensions == nil {
        return c.underlying.Apply(l)
    }
}

// follower 节点同步数据
func (c *FSM) Apply(log *raft.Log) interface{} {
    // 消息类型
    msgType := structs.MessageType(buf[0])

    // 找到消息对应的业务处理handler 进行处理
    if fn := c.apply[msgType]; fn != nil {
        return fn(buf[1:], log.Index)
    }
}

// structs.KVSRequestType 类型消息的处理handler
func (c *FSM) applyKVSOperation(buf []byte, index uint64) interface{} {
    var req structs.KVSRequest
    err := structs.Decode(buf, &req)

    switch req.Op {
    case api.KVSet:
        return c.state.KVSSet(index, &req.DirEnt)
    }
}

// KVSSet 存储键值信息
func (s *Store) KVSSet(idx uint64, entry *structs.DirEntry) error {
    tx := s.db.WriteTxn(idx)

    s.kvsSetTxn(tx, idx, entry, false)

    return tx.Commit()
}
```


## 参考资料

- [基于hashicorp/raft的分布式一致性实战教学    ](https://cloud.tencent.com/developer/article/1211094)

