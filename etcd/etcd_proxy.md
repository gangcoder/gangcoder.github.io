<!-- ---
title: etcd proxy
date: 2019-08-06 15:16:32
category: showcode, etcd
--- -->

# etcd proxy 实现

> etcd 代理安装在本地，将本地客户端的请求转发到etcd 服务端。

## main 入口

```go
// go.etcd.io/etcd/main.go
// main.go 引入etcdmain 包
import "go.etcd.io/etcd/etcdmain"

func main() {
	etcdmain.Main()
}

func Main() {
	if len(os.Args) > 1 {
		cmd := os.Args[1]
		// ...
		switch cmd {
        case "gateway", "grpc-proxy":
            // 执行终端代理命令
            // rootCmd.Execute 根据终端参数找到子命令，执行子命令的Run 函数
			if err := rootCmd.Execute(); err != nil {
				fmt.Fprint(os.Stderr, err)
				os.Exit(1)
			}
			return
		}
	}
}
```

## 注册命令

引入 `etcdmain` 包时，执行`init` 初始化函数。

```go
// go.etcd.io/etcd/etcdmain/gateway.go
// 初始化etcd 根命令
var (
	rootCmd = &cobra.Command{
		Use:        "etcd",
		Short:      "etcd server",
		SuggestFor: []string{"etcd"},
	}
)

// 注册gateway 子命令
func init() {
	rootCmd.AddCommand(newGatewayCommand())
}

func newGatewayCommand() *cobra.Command {
	lpc := &cobra.Command{
		Use:   "gateway <subcommand>",
		Short: "gateway related command",
	}
	lpc.AddCommand(newGatewayStartCommand())

	return lpc
}

// 注册gateway start 子命令
func newGatewayStartCommand() *cobra.Command {
	cmd := cobra.Command{
		Use:   "start",
		Short: "start the gateway",
		Run:   startGateway,
	}

    // ...
	cmd.Flags().StringVar(&gatewayListenAddr, "listen-addr", "127.0.0.1:23790", "listen address")
	cmd.Flags().StringSliceVar(&gatewayEndpoints, "endpoints", []string{"127.0.0.1:2379"}, "comma separated etcd cluster endpoints")

    return &cmd
}

// startGateway 实现逻辑
func startGateway(cmd *cobra.Command, args []string) {
	// 发现远端服务端点
	srvs := discoverEndpoints(lg, gatewayDNSCluster, gatewayCA, gatewayInsecureDiscovery, gatewayDNSClusterServiceName)
	if len(srvs.Endpoints) == 0 {
		// 如果没有找到服务端就用本地设置的默认值
		srvs.Endpoints = gatewayEndpoints
	}
	// 处理远端服务端点
	srvs.Endpoints = stripSchema(srvs.Endpoints)
	if len(srvs.SRVs) == 0 {
		for _, ep := range srvs.Endpoints {
			h, p, serr := net.SplitHostPort(ep)
			// ...
			var port uint16
			fmt.Sscanf(p, "%d", &port)
			srvs.SRVs = append(srvs.SRVs, &net.SRV{Target: h, Port: port})
		}
	}

    // 监听本地端口，接收请求
	var l net.Listener
	l, err = net.Listen("tcp", gatewayListenAddr)
    
    // tcp 代理服务
	tp := tcpproxy.TCPProxy{
		Logger:          lg,
		Listener:        l,
		Endpoints:       srvs.SRVs,
		MonitorInterval: getewayRetryDelay,
	}

    // 运行代理
	tp.Run()
}
```

## 运行代理服务

```go
// go.etcd.io/etcd/proxy/tcpproxy/userspace.go
type TCPProxy struct {
    // ...
	Listener        net.Listener
	Endpoints       []*net.SRV
	MonitorInterval time.Duration

	donec chan struct{}

	mu        sync.Mutex // guards the following fields
	remotes   []*remote
	pickCount int // for round robin
}

// 运行代理服务
func (tp *TCPProxy) Run() error {
    // ...
    // 将远端服务端点格式化为 remotes
	for _, srv := range tp.Endpoints {
		addr := fmt.Sprintf("%s:%d", srv.Target, srv.Port)
		tp.remotes = append(tp.remotes, &remote{srv: srv, addr: addr})
	}

    // 运行监控程序，判断远端服务是否正常
	go tp.runMonitor()
	for {
        // 监听本地请求
        in, err := tp.Listener.Accept()

		go tp.serve(in)
	}
}

// 处理代理请求
func (tp *TCPProxy) serve(in net.Conn) {
    // ...
	for {
		tp.mu.Lock()
        // 选择一个远端服务
        remote := tp.pick()
		tp.mu.Unlock()

		// 开启到远端服务的网络连接
		out, err = net.Dial("tcp", remote.addr)
        // ...
	}

    // 将代理的远端响应数据写入请求方
	go func() {
		io.Copy(in, out)
		in.Close()
		out.Close()
	}()

    // 将请求方请求写入代理的远端服务
	io.Copy(out, in)
	out.Close()
	in.Close()
}
```

## 参考资料

- go.etcd.io/etcd > etcdmain/gateway.go
