---
title: etcd KVServer grpc 服务
date: 2019-08-07 14:24:58
category: showcode, etcd
---

# etcd KVServer grpc 服务

> etcd 服务端grpc 接口实现逻辑。

## KVServer grpc 服务注册

grpc `KVServer` 服务注册逻辑。

```go
// go.etcd.io/etcd/etcdserver/api/v3rpc/quota.go
type quotaKVServer struct {
	pb.KVServer
	qa quotaAlarmer
}

func NewQuotaKVServer(s *etcdserver.EtcdServer) pb.KVServer {
	return &quotaKVServer{
		NewKVServer(s),
		quotaAlarmer{etcdserver.NewBackendQuota(s, "kv"), s, s.ID()},
	}
}

// go.etcd.io/etcd/etcdserver/api/v3rpc/key.go
type kvServer struct {
	hdr header
	kv  etcdserver.RaftKV
	// maxTxnOps is the max operations per txn.
	// e.g suppose maxTxnOps = 128.
	// Txn.Success can have at most 128 operations,
	// and Txn.Failure can have at most 128 operations.
	maxTxnOps uint
}

func NewKVServer(s *etcdserver.EtcdServer) pb.KVServer {
	return &kvServer{hdr: newHeader(s), kv: s, maxTxnOps: s.Cfg.MaxTxnOps}
}
```

## Get 请求处理

```go
// 键值Get 请求最终在这里处理
func (s *kvServer) Range(ctx context.Context, r *pb.RangeRequest) (*pb.RangeResponse, error) {
	if err := checkRangeRequest(r); err != nil {
		return nil, err
	}

	resp, err := s.kv.Range(ctx, r)
	if err != nil {
		return nil, togRPCError(err)
	}

	s.hdr.fill(resp.Header)
	return resp, nil
}
```

## 参考资料

- go.etcd.io/etcd > etcdserver/api/v3rpc/key.go

