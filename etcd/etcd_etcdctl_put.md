<!-- ---
title: etcd etcdctl put & get
date: 2019-08-04 04:35:53
category: showcode, etcd
--- -->

# etcdctl put 子命令

> etcdctl put 命令从etcd 中查询指定key 的value。

## Put 子命令

```go
// go.etcd.io/etcd/etcdctl/ctlv3/command/put_command.go
// NewPutCommand returns the cobra command for "put".
func NewPutCommand() *cobra.Command {
	cmd := &cobra.Command{
		Use:   "put [options] <key> <value> (<value> can also be given from stdin)",
		// ...
		Run: putCommandFunc,
	}
	cmd.Flags().StringVar(&leaseStr, "lease", "0", "lease ID (in hexadecimal) to attach to the key")
	// ...

	return cmd
}

// putCommandFunc executes the "put" command.
func putCommandFunc(cmd *cobra.Command, args []string) {
	key, value, opts := getPutOp(args)

	ctx, cancel := commandCtx(cmd)
	resp, err := mustClientFromCmd(cmd).Put(ctx, key, value, opts...)
	cancel()
	// ...
	display.Put(*resp)
}
```

## KV

```go
// 创建Client 时，键值对操作从这里继承
func NewKV(c *Client) KV {
	api := &kv{remote: RetryKVClient(c)}
	// ...
	return api
}

// mustClientFromCmd(cmd).Put(ctx, key, value, opts...) 实际执行函数
func (kv *kv) Put(ctx context.Context, key, val string, opts ...OpOption) (*PutResponse, error) {
	r, err := kv.Do(ctx, OpPut(key, val, opts...))
	return r.put, toErr(ctx, err)
}

// 转发请求
func (kv *kv) Do(ctx context.Context, op Op) (OpResponse, error) {
	var err error
	switch op.t {
	// ...
	case tPut:
		var resp *pb.PutResponse
		r := &pb.PutRequest{Key: op.key, Value: op.val, Lease: int64(op.leaseID), PrevKv: op.prevKV, IgnoreValue: op.ignoreValue, IgnoreLease: op.ignoreLease}
		resp, err = kv.remote.Put(ctx, r, kv.callOpts...)
		if err == nil {
			return OpResponse{put: (*PutResponse)(resp)}, nil
		}
	// ...
	default:
		panic("Unknown op")
	}
	return OpResponse{}, toErr(ctx, err)
}

// RetryKVClient Put 函数将请求通过grpc 发送到服务端
func (rkv *retryKVClient) Put(ctx context.Context, in *pb.PutRequest, opts ...grpc.CallOption) (resp *pb.PutResponse, err error) {
	return rkv.kc.Put(ctx, in, opts...)
}
```

## 参考资料

- go.etcd.io/etcd > clientv3/kv.go

