<!-- ---
title: minio server start
date: 2018-10-22 14:46:34
category: database, storage, minio
--- -->

minio server start

minio 服务启动实现

命令

```
minio server /data
```

分层实现

1. 启动minio cli 逻辑实现
    1. 创建cli app
    2. 注册子命令
    3. 运行cli app
    4. 根据终端参数启动服务
2. serverCmd 子命令实现
    1. 启动http server
    2. 注册api 处理函数
3. 针对具体请求处理

## minio 程序实现


```go
//minio/main.go
func main() {
	minio.Main(os.Args)
}

//处理minio 启动逻辑：创建app，并且启动app
//cmd/main.go
func Main(args []string) {
	// Set the minio app name.
	appName := filepath.Base(args[0])

	// Run the app - exit on error.
	if err := newApp(appName).Run(args); err != nil {
		os.Exit(1)
	}
}
```

### 创建App

实现两部分功能：

1. 注册处理cmd
2. 新建app

```go
//newApp
func newApp(name string) *cli.App {
	// Collection of minio commands currently supported are.
	commands := []cli.Command{}
	// registerCommand registers a cli command.
	registerCommand := func(command cli.Command) {
		commands = append(commands, command)
		commandsTree.Insert(command.Name)
	}

	//注册子命令
	registerCommand(serverCmd) //注册服务Cmd
	registerCommand(gatewayCmd) //网关
	registerCommand(updateCmd) //更新
	registerCommand(versionCmd) //版本控制

	app := cli.NewApp()
	app.Name = name
	app.Author = "Minio.io"
	app.Commands = commands
	app.CustomAppHelpTemplate = minioHelpTemplate

	return app
}

// NewApp 创建一个cli 应用
func NewApp() *App {
	return &App{
		Name:         filepath.Base(os.Args[0]),
		Action:       helpCommand.Action,
	}
}
```

### 启动app

Main 逻辑中，在创建app 后，会运行app

minio/cli/app.go

```go
//Run 是cli 应用的入口，解析终端参数并且交给具体子命令执行
func (a *App) Run(arguments []string) (err error) {
	//命令设置
	a.Setup()

	//补全命令的参数设置
	shellComplete, arguments := checkShellCompleteFlag(a, arguments)

	// parse flags
	set, err := flagSet(a.Name, a.Flags)
	if err != nil {
		return err
	}

	//解析配置
	set.SetOutput(ioutil.Discard)
	err = set.Parse(arguments[1:])
	
	if checkCompletions(context) {
		return nil
	}

	//根据终端参数，执行命令，对于minio 服务端，就是serverCmd
	args := context.Args()
	if args.Present() {
		name := args.First()
		c := a.Command(name)
		if c != nil {
			return c.Run(context)
		}
	}

	if a.Action == nil {
		a.Action = helpCommand.Action
	}

	// Run default Action
	err = HandleAction(a.Action, context)

	HandleExitCoder(err)
	return err
}
```

## Command

`minio/cli/command.go`

Command 结构体

## serverCmd 子命令实现

`minio/cmd/server-main.go`

serverCmd 是minio 服务的主服务，处理文件操作。

启动minio 服务时，执行`minio server /data`，minio 程序调用 serverCmd 子命令启动server。子程序再执行`c.Run(context)`。

整体实现分为

1. serverCmd 结构体
2. serverMain 逻辑
3. 配置http Handler
4. 注册api 路由

### serverCmd 结构体

serverCmd 子命令实现基于Command 结构体

1. 基于Command 结构体和 Run 启动serverCmd 的Action，也就是 `serverMain`
2. 执行 serverMain 逻辑：

启动serverCmd 的Action

```go
//创建cli 命令
//Action 指定了命令主逻辑
var serverCmd = cli.Command{
	Name:   "server",
	Usage:  "Start object storage server.",
	Flags:  append(serverFlags, globalFlags...),
	Action: serverMain,
}

//主cli 中根据参数名，启动子cli 的 c.Run(context) 函数
//Run 命令调用的是 minio/cli/command.go 中Command 结构体的 Run 命令
//解析ctx 中的参数，并且生成子命令的参数
func (c Command) Run(ctx *Context) (err error) {
	if len(c.Subcommands) > 0 {
		return c.startApp(ctx)
	}

	//解析终端参数
	set, err := flagSet(c.Name, c.Flags)
	if err != nil {
		return err
	}
	set.SetOutput(ioutil.Discard)

	context := NewContext(ctx.App, set, ctx)
	if checkCommandCompletions(context, c.Name) {
		return nil
	}

	//解析帮助函数命令
	if checkCommandHelp(context, c.Name) {
		return nil
	}

	//当没有设置Action 时，附加Help 命令的Action
	if c.Action == nil {
		c.Action = helpSubcommand.Action
	}

	//运行Action
	context.Command = c
	err = HandleAction(c.Action, context) //对于serverCmd，Action 是serverMain

	return err
}

//这里Action 是serverMain
//HandleAction 判断Action 类型，并且执行Action
func HandleAction(action interface{}, context *Context) (err error) {
	if a, ok := action.(ActionFunc); ok {
		return a(context)
	} else if a, ok := action.(func(*Context) error); ok {
		return a(context)
	} else if a, ok := action.(func(*Context)); ok { // deprecated function signature
		a(context)
		return nil
	} else {
		return errInvalidActionType
	}
}
```

### serverMain 逻辑

执行 serverCmd 中的serverMain 逻辑

serverMain 是minio 文件服务的主逻辑函数，主要实现的逻辑包括：

1. 处理终端和环境变量参数初始化参数
2. 配置http 服务监听，注册http 接口路由，启动http 服务
3. 创建对象处理层和缓存处理层
4. 创建和初始化子系统IAM, policy, notification

```go
//rverMain handler called for 'minio server' command.
func serverMain(ctx *cli.Context) {
	if ctx.Args().First() == "help" || !endpointsPresent(ctx) {
		cli.ShowCommandHelpAndExit(ctx, "server", 1)
	}

	// Disable logging until server initialization is complete, any
	// error during initialization will be shown as a fatal message
	logger.Disable = true

	//处理终端命令参数
	serverHandleCmdArgs(ctx)

	//处理环境变量参数
	serverHandleEnvVars()

	//检查更新
	if !quietFlag {
		// Check for new updates from dl.minio.io.
		mode := globalMinioModeFS
		if globalIsDistXL {
			mode = globalMinioModeDistXL
		} else if globalIsXL {
			mode = globalMinioModeXL
		}
		checkUpdate(mode)
	}

	// Set system resources to maximum.
	logger.LogIf(context.Background(), setMaxResources())

	//设置分布式系统中节点间内容的同步
	if globalIsDistXL {
		globalDsync, err = dsync.New(newDsyncNodes(globalEndpoints))
		if err != nil {
			logger.Fatal(err, "Unable to initialize distributed locking on %s", globalEndpoints)
		}
	}

	// Initialize name space lock.
	initNSLock(globalIsDistXL)

	// Init global heal state
	initAllHealState(globalIsXL)

	//配置http 服务监听，注册http 接口路由
	var handler http.Handler
	handler, err = configureServerHandler(globalEndpoints)
	if err != nil {
		logger.Fatal(uiErrUnexpectedError(err), "Unable to configure one of server's RPC services")
	}

	// Initialize Admin Peers inter-node communication only in distributed setup.
	initGlobalAdminPeers(globalEndpoints)

	//更新全局http 服务
	globalHTTPServer = xhttp.NewServer([]string{globalMinioAddr}, criticalErrorHandler{handler}, getCert)
	globalHTTPServer.UpdateBytesReadFunc = globalConnStats.incInputBytes
	globalHTTPServer.UpdateBytesWrittenFunc = globalConnStats.incOutputBytes
	go func() {
		//开启http 服务
		globalHTTPServerErrorCh <- globalHTTPServer.Start()
	}()

	//监听终端消息
	signal.Notify(globalOSSignalCh, os.Interrupt, syscall.SIGTERM)

	//新建对象处理层
	newObject, err := newObjectLayer(globalEndpoints)
	
	// Populate existing buckets to the etcd backend
	if globalDNSConfig != nil {
		initFederatorBackend(newObject)
	}

	// Re-enable logging
	logger.Disable = false

	// Create a new config system.
	globalConfigSys = NewConfigSys()

	// Initialize config system.
	if err = globalConfigSys.Init(newObject); err != nil {
		logger.Fatal(err, "Unable to initialize config system")
	}

	// Load logger subsystem
	loadLoggers()

	//获取缓存处理层配置
	var cacheConfig = globalServerConfig.GetCacheConfig()
	if len(cacheConfig.Drives) > 0 {
		//初始化磁盘缓存对象， 用于文件处理缓存逻辑
		globalCacheObjectAPI, err = newServerCacheObjects(cacheConfig)
		logger.FatalIf(err, "Unable to initialize disk caching")
	}

	//创建和初始化子系统IAM, policy, notification
	// Create new IAM system.
	globalIAMSys = NewIAMSys()
	if err = globalIAMSys.Init(newObject); err != nil {
		logger.Fatal(err, "Unable to initialize IAM system")
	}

	// Create new policy system.
	globalPolicySys = NewPolicySys()

	// Initialize policy system.
	if err = globalPolicySys.Init(newObject); err != nil {
		logger.Fatal(err, "Unable to initialize policy system")
	}

	// Create new notification system.
	globalNotificationSys = NewNotificationSys(globalServerConfig, globalEndpoints)

	// Initialize notification system.
	if err = globalNotificationSys.Init(newObject); err != nil {
		logger.Fatal(err, "Unable to initialize notification system")
	}

	globalObjLayerMutex.Lock()
	globalObjectAPI = newObject
	globalObjLayerMutex.Unlock()

	//打印启动信息
	apiEndpoints := getAPIEndpoints(globalMinioAddr)
	printStartupMessage(apiEndpoints)

	//设置在线时间
	globalBootTime = UTCNow()

	handleSignals()
}
```

### ObjectAPI 实现

在运行 serverMain 主逻辑时，会创建 ObjectAPI 和CacheAPI 操作层，然后在处理object 信息时，调用ObjectAPI 和CacheAPI 操作层的函数实现具体操作。

```go
//server-main.go
newObject, err := newObjectLayer(globalEndpoints)

//server-main.go
globalObjectAPI = newObject

// Initialize object layer with the supplied disks, objectLayer is nil upon any error.
func newObjectLayer(endpoints EndpointList) (newObject ObjectLayer, err error) {
	// For FS only, directly use the disk.

	isFS := len(endpoints) == 1
	if isFS {
		// Initialize new FS object layer.
		return NewFSObjectLayer(endpoints[0].Path)
	}

	format, err := waitForFormatXL(context.Background(), endpoints[0].IsLocal, endpoints, globalXLSetCount, globalXLSetDriveCount)

	return newXLSets(endpoints, format, len(format.XL.Sets), len(format.XL.Sets[0]))
}
```

### 配置http Handler

serverCmd 中的http 处理逻辑

```go
//配置http Handler
handler, err = configureServerHandler(globalEndpoints)

//启动http 服务
globalHTTPServer.Start()
```

configureServerHandler 实现在 `minio/cmd/routers.go` 中，主要是添加http handler 路由注册逻辑。

```go
//configureServer 处理http　服务的路由注册
func configureServerHandler(endpoints EndpointList) (http.Handler, error) {
	// Initialize distributed NS lock.
	if globalIsDistXL {
		registerDistXLRouters(router, endpoints)
	}

	// Add Admin RPC router
	registerAdminRPCRouter(router)

	// Add Admin router.
	registerAdminRouter(router)

	// Add healthcheck router
	registerHealthCheckRouter(router)

	// Add server metrics router
	registerMetricsRouter(router)

	// Register web router when its enabled.
	if globalIsBrowserEnabled {
		if err := registerWebRouter(router); err != nil {
			return nil, err
		}
	}

	// Add API router.
	//添加文件处理api 路由
	registerAPIRouter(router)

	// Register rest of the handlers.
	return registerHandlers(router, globalHandlers...), nil
}
```

### 添加文件处理api 路由

添加处理bucket 和文件对象的api 路由，是minio 文件服务的主要接口。

`minio/cmd/api-router.go`

api 处理函数分为：bucket 处理，object 处理。

bucket 处理在`minio/cmd/bucket-handlers.go` 中实现，object 处理在 `minio/cmd/object-handlers.go` 中实现，最终都是通过调用ObjectAPI 的函数完成处理。


```go
//api-router.go
//registerAPIRouter 配置http Handler，兼容 S3 格式
func registerAPIRouter(router *mux.Router) {
	// Initialize API.
	api := objectAPIHandlers{
		ObjectAPI: newObjectLayerFn,
		CacheAPI:  newCacheObjectsFn,
	}

	// API Router
	//根据bucket 添加子路由
	apiRouter := router.PathPrefix("/").Subrouter()
	var routers []*mux.Router
	if globalDomainName != "" {
		routers = append(routers, apiRouter.Host("{bucket:.+}."+globalDomainName).Subrouter())
	}
	routers = append(routers, apiRouter.PathPrefix("/{bucket}").Subrouter())

	//针对子路由，具体注册接口服务
	for _, bucket := range routers {
		// Object operations
		// CopyObjectPart
		bucket.Methods("PUT").Path("/{object:.+}").HeadersRegexp("X-Amz-Copy-Source", ".*?(\\/|%2F).*?").HandlerFunc(httpTraceAll(api.CopyObjectPartHandler)).Queries("partNumber", "{partNumber:[0-9]+}", "uploadId", "{uploadId:.*}")
		// PutObjectPart
		bucket.Methods("PUT").Path("/{object:.+}").HandlerFunc(httpTraceHdrs(api.PutObjectPartHandler)).Queries("partNumber", "{partNumber:[0-9]+}", "uploadId", "{uploadId:.*}")
		// GetObject
		bucket.Methods("GET").Path("/{object:.+}").HandlerFunc(httpTraceHdrs(api.GetObjectHandler))
		// CopyObject
		bucket.Methods("PUT").Path("/{object:.+}").HeadersRegexp("X-Amz-Copy-Source", ".*?(\\/|%2F).*?").HandlerFunc(httpTraceAll(api.CopyObjectHandler))
		// PutObject
		bucket.Methods("PUT").Path("/{object:.+}").HandlerFunc(httpTraceHdrs(api.PutObjectHandler))
		// DeleteObject
		bucket.Methods("DELETE").Path("/{object:.+}").HandlerFunc(httpTraceAll(api.DeleteObjectHandler))

		/// Bucket operations
		// GetBucketLocation
		bucket.Methods("GET").HandlerFunc(httpTraceAll(api.GetBucketLocationHandler)).Queries("location", "")
		// PutBucket
		bucket.Methods("PUT").HandlerFunc(httpTraceAll(api.PutBucketHandler))
	}

	/// Root operation

	// ListBuckets
	apiRouter.Methods("GET").Path("/").HandlerFunc(httpTraceAll(api.ListBucketsHandler))

	// If none of the routes match.
	apiRouter.NotFoundHandler = http.HandlerFunc(httpTraceAll(notFoundHandler))
}
```

### 运行http 服务

server-main 中开启http 服务

minio/cmd/http/server.go

```go
//server-main.go
globalHTTPServerErrorCh <- globalHTTPServer.Start()

//minio/cmd/http/server.go
// Start - start HTTP server
func (srv *Server) Start() (err error) {
	readTimeout := srv.ReadTimeout
	writeTimeout := srv.WriteTimeout
	handler := srv.Handler // if srv.Handler holds non-synced state -> possible data race

	addrs := set.CreateStringSet(srv.Addrs...).ToSlice() // copy and remove duplicates
	tcpKeepAliveTimeout := srv.TCPKeepAliveTimeout
	updateBytesReadFunc := srv.UpdateBytesReadFunc
	updateBytesWrittenFunc := srv.UpdateBytesWrittenFunc

	// Create new HTTP listener.
	var listener *httpListener
	listener, err = newHTTPListener(
		addrs,
		tlsConfig,
		tcpKeepAliveTimeout,
		readTimeout,
		writeTimeout,
		srv.MaxHeaderBytes,
		updateBytesReadFunc,
		updateBytesWrittenFunc,
	)

	// Wrap given handler to do additional
	wrappedHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		atomic.AddInt32(&srv.requestCount, 1)
		defer atomic.AddInt32(&srv.requestCount, -1)

		// Handle request using passed handler.
		handler.ServeHTTP(w, r)
	})

	srv.listenerMutex.Lock()
	srv.Handler = wrappedHandler
	srv.listener = listener
	srv.listenerMutex.Unlock()

	//开启http 服务
	return srv.Server.Serve(listener)
}
```

## gatewayCmd

## updateCmd

## versionCmd



## 参考

- [https://github.com/minio/minio/blob/master/main.go](https://github.com/minio/minio/blob/master/main.go)

