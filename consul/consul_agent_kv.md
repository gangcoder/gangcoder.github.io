<!-- ---
title: agent kv
date: 2019-07-29 20:02:35
category: showcode, consul
--- -->

agent kv

## 注册grpc 服务

```go
// github.com/hashicorp/consul/agent/consul/server_oss.go
registerEndpoint(func(s *Server) interface{} { return &KVS{s, s.loggers.Named(logging.KV)} })
```

```go
type KVS struct {
	srv    *Server
	logger hclog.Logger
}
```

## 数据查询

```go
// Get is used to lookup a single key.
func (k *KVS) Get(args *structs.KeyRequest, reply *structs.IndexedDirEntries) error {
    // 是否远程处理
	if done, err := k.srv.forward("KVS.Get", args, args, reply); done {
		return err
	}

    // 从state 中查询数据
	return k.srv.blockingQuery(
		&args.QueryOptions,
		&reply.QueryMeta,
		func(ws memdb.WatchSet, state *state.Store) error {
			index, ent, err := state.KVSGet(ws, args.Key, &args.EnterpriseMeta)
			if err != nil {
				return err
            }

            reply.Index = ent.ModifyIndex
			reply.Entries = structs.DirEntries{ent}
			// ...
			return nil
		})
}
```

### 请求远程服务

```go
func (s *Server) forward(method string, info structs.RPCInfo, args interface{}, reply interface{}) (bool, error) {
	// 分配数据中心处理
	dc := info.RequestDatacenter()
	if dc != s.config.Datacenter {
        // ...
		err := s.forwardDC(method, dc, args, reply)
		return true, err
	}

	// 查找主节点
	isLeader, leader := s.getLeader()

	// 如果当前节点是主节点，则返回交给主节点处理
	if isLeader {
		return false, nil
	}

        // 否则，远程调用
    rpcErr = s.connPool.RPC(s.config.Datacenter, leader.ShortName, leader.Addr,
        method, args, reply)
    
    return true, rpcErr
    }
}
```

### 数据查询

```go
func (s *Server) blockingQuery(queryOpts structs.QueryOptionsCompat, queryMeta structs.QueryMetaCompat, fn queryFn) error {
	// 状态管理器
	state := s.fsm.State()
    
    // ...
	// 执行查询
    err := fn(ws, state)
}
```

## 数据写入

```go
// Apply is used to apply a KVS update request to the data store.
func (k *KVS) Apply(args *structs.KVSRequest, reply *bool) error {
	if done, err := k.srv.forward("KVS.Apply", args, args, reply); done {
		return err
	}
	
	// 基于Raft 协议进行数据更新
	resp, err := k.srv.raftApply(structs.KVSRequestType, args)
    
    // ...
	if respErr, ok := resp.(error); ok {
		return respErr
	}
}
```

```go
func (s *Server) raftApply(t structs.MessageType, msg interface{}) (interface{}, error) {
	return s.raftApplyMsgpack(t, msg)
}

func (s *Server) raftApplyMsgpack(t structs.MessageType, msg interface{}) (interface{}, error) {
	return s.raftApplyWithEncoder(t, msg, structs.Encode)
}

func (s *Server) raftApplyWithEncoder(t structs.MessageType, msg interface{}, encoder raftEncoder) (interface{}, error) {
	// ...
	buf, err := encoder(t, msg)
    
    // ...
    s.raft.Apply(buf, enqueueLimit)
}

func (r *Raft) Apply(cmd []byte, timeout time.Duration) ApplyFuture {
	return r.ApplyLog(Log{Data: cmd}, timeout)
}

func (r *Raft) ApplyLog(log Log, timeout time.Duration) ApplyFuture {
	// ...
	// Create a log future, no index or term yet
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
		return logFuture
	}
}
```

## 参考资料

- github.com/hashicorp/consul/agent/consul/kvs_endpoint.go

