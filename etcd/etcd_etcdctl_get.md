<!-- ---
title: etcd etcdctl put & get
date: 2019-08-04 04:35:53
category: showcode, etcd
--- -->

# etcdctl get 子命令

> etcdctl get 命令从etcd 中查询指定key 的value。

## Get 子命令

```go
// go.etcd.io/etcd/etcdctl/ctlv3/ctl.go
// 添加子命令
rootCmd.AddCommand(
	command.NewGetCommand()
)

//go.etcd.io/etcd/etcdctl/ctlv3/command/get_command.go
// NewGetCommand 创建基于cobra 的get 子命令
func NewGetCommand() *cobra.Command {
	cmd := &cobra.Command{
		Use:   "get [options] <key> [range_end]",
		Run:   getCommandFunc,
	}

	cmd.Flags().BoolVar(&getKeysOnly, "keys-only", false, "Get only the keys")
	// ...

	return cmd
}

// getCommandFunc 执行get 操作主逻辑
// 创建etcdctl 到etcd 服务端的Client
// 发送Get 请求
// 展示Get 请求返回的结果
func getCommandFunc(cmd *cobra.Command, args []string) {
	key, opts := getGetOp(args)
	resp, err := mustClientFromCmd(cmd).Get(ctx, key, opts...)
	// ...
	display.Get(*resp)
}
```

## 创建Client

```go
// go.etcd.io/etcd/etcdctl/ctlv3/command/global.go
// 从终端获取参数
// 创建Client
func mustClientFromCmd(cmd *cobra.Command) *clientv3.Client {
	cfg := clientConfigFromCmd(cmd)
	return cfg.mustClient()
}

// clientConfigFromCmd 从终端获取参数，创建clientConfig 实例
type clientConfig struct {
	endpoints        []string
	dialTimeout      time.Duration
	keepAliveTime    time.Duration
	keepAliveTimeout time.Duration
	scfg             *secureCfg
	acfg             *authCfg
}

// 根据终端参数，创建Client
func (cc *clientConfig) mustClient() *clientv3.Client {
	cfg, err := newClientCfg(cc.endpoints, cc.dialTimeout, cc.keepAliveTime, cc.keepAliveTimeout, cc.scfg, cc.acfg)
	
	// 创建client
	client, err := clientv3.New(*cfg)

	return client
}

// go.etcd.io/etcd/clientv3/config.go
// newClientCfg 创建全局clientv3 配置实例
type Config struct {
	// 服务端端点
	Endpoints []string `json:"endpoints"`
	// ...
}

// go.etcd.io/etcd/clientv3/client.go
// Client 管理v3 客户端到etcd 服务端的连接会话
type Client struct {
	Cluster
	KV
	Lease
	Watcher
	Auth
	Maintenance

	conn *grpc.ClientConn
	cfg           Config
	resolverGroup *endpoint.ResolverGroup
	// ...
}

// 创建Client
func New(cfg Config) (*Client, error) {
	// ...
	return newClient(&cfg)
}

func newClient(cfg *Config) (*Client, error) {
	// ...
	client := &Client{
		conn:     nil,
		cfg:      *cfg,
		creds:    creds,
		ctx:      ctx,
		cancel:   cancel,
		mu:       new(sync.RWMutex),
		callOpts: defaultCallOpts,
	}

	// 准备到etcd 端点的解析器
	client.resolverGroup, err = endpoint.NewResolverGroup(fmt.Sprintf("client-%s", uuid.New().String()))
	client.resolverGroup.SetEndpoints(cfg.Endpoints)
	// ...
	dialEndpoint := cfg.Endpoints[0]

	// 创建到etcd 服务端的grpc 连接
	conn, err := client.dialWithBalancer(dialEndpoint, grpc.WithBalancerName(roundRobinBalancerName))
	client.conn = conn

	// 通过继承，实现client 的集群，键值对，租约等接口
	client.Cluster = NewCluster(client)
	client.KV = NewKV(client)
	client.Lease = NewLease(client)
	client.Watcher = NewWatcher(client)
	client.Auth = NewAuth(client)
	client.Maintenance = NewMaintenance(client)

	// 同步etcd 集群端点信息
	go client.autoSync()
	return client, nil
}
```

## NewKV

`NewKV` 返回实现键值对操作的实例对象，Client 通过继承该对象，实现`kv` 接口，这样可以在Client 上直接调用`kv` 操作函数。

```go
// kv 对象
type kv struct {
	remote   pb.KVClient
	callOpts []grpc.CallOption
}

type KV interface {
	Put(ctx context.Context, key, val string, opts ...OpOption) (*PutResponse, error)
	Get(ctx context.Context, key string, opts ...OpOption) (*GetResponse, error)
	Delete(ctx context.Context, key string, opts ...OpOption) (*DeleteResponse, error)
	Compact(ctx context.Context, rev int64, opts ...CompactOption) (*CompactResponse, error)
	Do(ctx context.Context, op Op) (OpResponse, error)
	Txn(ctx context.Context) Txn
}

func NewKV(c *Client) KV {
	api := &kv{remote: RetryKVClient(c)}
	if c != nil {
		api.callOpts = c.callOpts
	}
	return api
}

// go.etcd.io/etcd/clientv3/retry.go
// RetryKVClient 封装了grpc Client 实例
func RetryKVClient(c *Client) pb.KVClient {
	return &retryKVClient{
		kc: pb.NewKVClient(c.conn),
	}
}

// mustClientFromCmd(cmd).Get(ctx, key, opts...) 请求到此处
func (kv *kv) Get(ctx context.Context, key string, opts ...OpOption) (*GetResponse, error) {
	r, err := kv.Do(ctx, OpGet(key, opts...))
	return r.get, toErr(ctx, err)
}

func (kv *kv) Do(ctx context.Context, op Op) (OpResponse, error) {
	var err error
	switch op.t {
	case tRange:
		var resp *pb.RangeResponse
		resp, err = kv.remote.Range(ctx, op.toRangeRequest(), kv.callOpts...)
		if err == nil {
			return OpResponse{get: (*GetResponse)(resp)}, nil
		}
	// ...
	default:
		panic("Unknown op")
	}
	return OpResponse{}, toErr(ctx, err)
}

// remote 是RetryKVClient 实例，因此请求到此处
// 这里最终请求grpc 服务端接口
func (rkv *retryKVClient) Range(ctx context.Context, in *pb.RangeRequest, opts ...grpc.CallOption) (resp *pb.RangeResponse, err error) {
	return rkv.kc.Range(ctx, in, append(opts, withRetryPolicy(repeatable))...)
}
```


## 参考资料

- go.etcd.io/etcd > clientv3/kv.go

