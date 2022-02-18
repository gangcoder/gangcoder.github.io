<!-- ---
title: etcd etcd
date: 2019-08-04 04:43:44
category: showcode, etcd
--- -->

# etcd 服务启动实现

> etcd 服务端启动逻辑

## main 入口函数

```go
// go.etcd.io/etcd/main.go
// main.go 入口函数
func main() {
	etcdmain.Main()
}

// go.etcd.io/etcd/etcdmain/main.go
func Main() {
	// ...
	// 启动etcd 服务
	startEtcdOrProxyV2()
}
```

```go
// go.etcd.io/etcd/etcdmain/etcd.go
func startEtcdOrProxyV2() {
	grpc.EnableTracing = false

	// 处理默认配置
	cfg := newConfig()
	defaultInitialCluster := cfg.ec.InitialCluster

	// 解析配置
	err := cfg.parse(os.Args[1:])
	// ...

	// 集群信息
	defaultHost, dhErr := (&cfg.ec).UpdateDefaultClusterFromName(defaultInitialCluster)

	// 处理配置目录信息
	if cfg.ec.Dir == "" {
		cfg.ec.Dir = fmt.Sprintf("%v.etcd", cfg.ec.Name)
		// ...
	}

	var stopped <-chan struct{}
	var errc <-chan error

	// 监察配置目录
	which := identifyDataDirOrDie(cfg.ec.GetLogger(), cfg.ec.Dir)
	if which != dirEmpty {
		// 如果是成员配置，就开始etcd
		// 如果是代理配置，就开启代理
		switch which {
		case dirMember:
			stopped, errc, err = startEtcd(&cfg.ec)
		case dirProxy:
			err = startProxy(cfg)
		default:
			// ...
		}
	} else {
		// 如果没有查到配置文件信息
		// 就从终端参数中获取信息
		shouldProxy := cfg.isProxy()
		if !shouldProxy {
			// 不需要开启代理时，就启动etcd
			stopped, errc, err = startEtcd(&cfg.ec)
			// ...
		}
		// 开启代理
		if shouldProxy {
			err = startProxy(cfg)
		}
	}

	// 处理终端中断信息
	osutil.HandleInterrupts(lg)

	// 通知Systemd 
	notifySystemd(lg)

	// ...
	osutil.Exit(0)
}
```

## 启动etcd 服务

```go
// go.etcd.io/etcd/etcdmain/etcd.go
// startEtcd 启动etcd 服务
func startEtcd(cfg *embed.Config) (<-chan struct{}, <-chan error, error) {
	e, err := embed.StartEtcd(cfg)
	
	// 注册终端处理函数
	osutil.RegisterInterruptHandler(e.Close)
	select {
	case <-e.Server.ReadyNotify(): // wait for e.Server to join the cluster
	case <-e.Server.StopNotify(): // publish aborted from 'ErrStopped'
	}
	return e.Server.StopNotify(), e.Err(), nil
}

// go.etcd.io/etcd/embed/etcd.go
// StartEtcd etcd 启动逻辑
func StartEtcd(inCfg *Config) (e *Etcd, err error) {
	// ...
	serving := false
	e = &Etcd{cfg: *inCfg, stopc: make(chan struct{})}
	cfg := &e.cfg
	
	// 监听同伴节点信息
	if e.Peers, err = configurePeerListeners(cfg); err != nil {
		return e, err
	}

	// 开启客户端监听
	if e.sctxs, err = configureClientListeners(cfg); err != nil {
		return e, err
	}

	for _, sctx := range e.sctxs {
		e.Clients = append(e.Clients, sctx.l)
	}

	var (
		urlsmap types.URLsMap
		token   string
	)
	memberInitialized := true
	if !isMemberInitialized(cfg) {
		memberInitialized = false
		urlsmap, token, err = cfg.PeerURLsMapAndToken("etcd")
		if err != nil {
			return e, fmt.Errorf("error setting up initial cluster: %v", err)
		}
	}

	// ...
	// etcd 服务配置
	srvcfg := etcdserver.ServerConfig{
		Name:                       cfg.Name,
		ClientURLs:                 cfg.ACUrls,
		PeerURLs:                   cfg.APUrls,
		DataDir:                    cfg.Dir,
	}

	// 创建etcd 服务
	if e.Server, err = etcdserver.NewServer(srvcfg); err != nil {
		return e, err
	}

	// 运行服务
	e.Server.Start()

	if err = e.servePeers(); err != nil {
		return e, err
	}
	// etcd 服务逻辑，处理etcd 客户端请求
	if err = e.serveClients(); err != nil {
		return e, err
	}
	if err = e.serveMetrics(); err != nil {
		return e, err
	}
	// ...
	serving = true
	return e, nil
}
```

## etcd grpc 服务

```go
// go.etcd.io/etcd/embed/etcd.go
func (e *Etcd) serveClients() (err error) {
	// ...
	// Start a client server goroutine for each listen address
	var h http.Handler
	// ...
	// 调试信息
	mux := http.NewServeMux()
	etcdhttp.HandleBasic(mux, e.Server)
	h = mux

	// 开启客户端请求处理服务
	for _, sctx := range e.sctxs {
		go func(s *serveCtx) {
			e.errHandler(s.serve(e.Server, &e.cfg.ClientTLSInfo, h, e.errHandler, gopts...))
		}(sctx)
	}
	return nil
}

// go.etcd.io/etcd/embed/serve.go
// 服务处理接收请求，针对每个请求创建一个goroutine，接收请求参数，处理后返回
func (sctx *serveCtx) serve(
	s *etcdserver.EtcdServer,
	tlsinfo *transport.TLSInfo,
	handler http.Handler,
	errHandler func(error),
	gopts ...grpc.ServerOption) (err error) {

	// ...
	m := cmux.New(sctx.l)
	v3c := v3client.New(s)
	servElection := v3election.NewElectionServer(v3c)
	servLock := v3lock.NewLockServer(v3c)

	// ...
	// 非tls 情况下
	if sctx.insecure {
		// 注册grpc 服务
		gs = v3rpc.Server(s, nil, gopts...)
		v3electionpb.RegisterElectionServer(gs, servElection)
		v3lockpb.RegisterLockServer(gs, servLock)
		if sctx.serviceRegister != nil {
			sctx.serviceRegister(gs)
		}

		// 针对grpc 请求，开启grpc 服务
		grpcl := m.Match(cmux.HTTP2())
		go func() { errHandler(gs.Serve(grpcl)) }()

		// 开启grpc 网关
		var gwmux *gw.ServeMux
		if s.Cfg.EnableGRPCGateway {
			gwmux, err = sctx.registerGateway([]grpc.DialOption{grpc.WithInsecure()})
			// ...
		}

		// 开启http 服务
		httpmux := sctx.createMux(gwmux, handler)
		srvhttp := &http.Server{
			Handler:  createAccessController(sctx.lg, s, httpmux),
			ErrorLog: logger, // do not log user error
		}
		httpl := m.Match(cmux.HTTP1())
		go func() { errHandler(srvhttp.Serve(httpl)) }()

		sctx.serversC <- &servers{grpc: gs, http: srvhttp}
		// ...
	}

	// ...
	close(sctx.serversC)
	return m.Serve()
}

// go.etcd.io/etcd/etcdserver/api/v3rpc/grpc.go
// 注册grpc 服务端处理逻辑
func Server(s *etcdserver.EtcdServer, tls *tls.Config, gopts ...grpc.ServerOption) *grpc.Server {
	// ...
	opts = append(opts, grpc.MaxRecvMsgSize(int(s.Cfg.MaxRequestBytes+grpcOverheadBytes)))

	// 创建grpc 服务
	grpcServer := grpc.NewServer(append(opts, gopts...)...)

	// 注册grpc 服务
	pb.RegisterKVServer(grpcServer, NewQuotaKVServer(s))
	pb.RegisterWatchServer(grpcServer, NewWatchServer(s))
	pb.RegisterLeaseServer(grpcServer, NewQuotaLeaseServer(s))
	pb.RegisterClusterServer(grpcServer, NewClusterServer(s))
	pb.RegisterAuthServer(grpcServer, NewAuthServer(s))
	pb.RegisterMaintenanceServer(grpcServer, NewMaintenanceServer(s))

	// 注册健康服务
	hsrv := health.NewServer()
	hsrv.SetServingStatus("", healthpb.HealthCheckResponse_SERVING)
	healthpb.RegisterHealthServer(grpcServer, hsrv)

	// set zero values for metrics registered for this grpc server
	grpc_prometheus.Register(grpcServer)

	return grpcServer
}
```


## 参考资料

- go.etcd.io/etcd > main.go

