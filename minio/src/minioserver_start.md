---
title: minio server start
date: 2018-10-22 14:46:34
category: database, storage, minio
---

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
		// ListObjectPxarts
		bucket.Methods("GET").Path("/{object:.+}").HandlerFunc(httpTraceAll(api.ListObjectPartsHandler)).Queries("uploadId", "{uploadId:.*}")
		// CompleteMultipartUpload
		bucket.Methods("POST").Path("/{object:.+}").HandlerFunc(httpTraceAll(api.CompleteMultipartUploadHandler)).Queries("uploadId", "{uploadId:.*}")
		// NewMultipartUpload
		bucket.Methods("POST").Path("/{object:.+}").HandlerFunc(httpTraceAll(api.NewMultipartUploadHandler)).Queries("uploads", "")
		// AbortMultipartUpload
		bucket.Methods("DELETE").Path("/{object:.+}").HandlerFunc(httpTraceAll(api.AbortMultipartUploadHandler)).Queries("uploadId", "{uploadId:.*}")
		// GetObjectACL - this is a dummy call.
		bucket.Methods("GET").Path("/{object:.+}").HandlerFunc(httpTraceHdrs(api.GetObjectACLHandler)).Queries("acl", "")
		// SelectObjectContent
		bucket.Methods("POST").Path("/{object:.+}").HandlerFunc(httpTraceHdrs(api.SelectObjectContentHandler)).Queries("select", "").Queries("select-type", "2")
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
		// GetBucketPolicy
		bucket.Methods("GET").HandlerFunc(httpTraceAll(api.GetBucketPolicyHandler)).Queries("policy", "")
		// GetBucketACL -- this is a dummy call.
		bucket.Methods("GET").HandlerFunc(httpTraceAll(api.GetBucketACLHandler)).Queries("acl", "")
		// GetBucketNotification
		bucket.Methods("GET").HandlerFunc(httpTraceAll(api.GetBucketNotificationHandler)).Queries("notification", "")
		// ListenBucketNotification
		bucket.Methods("GET").HandlerFunc(httpTraceAll(api.ListenBucketNotificationHandler)).Queries("events", "{events:.*}")
		// ListMultipartUploads
		bucket.Methods("GET").HandlerFunc(httpTraceAll(api.ListMultipartUploadsHandler)).Queries("uploads", "")
		// ListObjectsV2
		bucket.Methods("GET").HandlerFunc(httpTraceAll(api.ListObjectsV2Handler)).Queries("list-type", "2")
		// ListObjectsV1 (Legacy)
		bucket.Methods("GET").HandlerFunc(httpTraceAll(api.ListObjectsV1Handler))
		// PutBucketPolicy
		bucket.Methods("PUT").HandlerFunc(httpTraceAll(api.PutBucketPolicyHandler)).Queries("policy", "")
		// PutBucketNotification
		bucket.Methods("PUT").HandlerFunc(httpTraceAll(api.PutBucketNotificationHandler)).Queries("notification", "")
		// PutBucket
		bucket.Methods("PUT").HandlerFunc(httpTraceAll(api.PutBucketHandler))
		// HeadBucket
		bucket.Methods("HEAD").HandlerFunc(httpTraceAll(api.HeadBucketHandler))
		// PostPolicy
		bucket.Methods("POST").HeadersRegexp("Content-Type", "multipart/form-data*").HandlerFunc(httpTraceHdrs(api.PostPolicyBucketHandler))
		// DeleteMultipleObjects
		bucket.Methods("POST").HandlerFunc(httpTraceAll(api.DeleteMultipleObjectsHandler)).Queries("delete", "")
		// DeleteBucketPolicy
		bucket.Methods("DELETE").HandlerFunc(httpTraceAll(api.DeleteBucketPolicyHandler)).Queries("policy", "")
		// DeleteBucket
		bucket.Methods("DELETE").HandlerFunc(httpTraceAll(api.DeleteBucketHandler))
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

## bucket/object 操作


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

//fs-v1.go
// NewFSObjectLayer - initialize new fs object layer.
func NewFSObjectLayer(fsPath string) (ObjectLayer, error) {
	ctx := context.Background()

	// Assign a new UUID for FS minio mode. Each server instance
	// gets its own UUID for temporary file transaction.
	fsUUID := mustGetUUID()

	// Initialize meta volume, if volume already exists ignores it.
	if err = initMetaVolumeFS(fsPath, fsUUID); err != nil {
		return nil, err
	}

	// Initialize `format.json`, this function also returns.
	rlk, err := initFormatFS(ctx, fsPath)
	if err != nil {
		return nil, err
	}

	// Initialize fs objects.
	fs := &FSObjects{
		fsPath:       fsPath,
		metaJSONFile: fsMetaJSONFile,
		fsUUID:       fsUUID,
		rwPool: &fsIOPool{
			readersMap: make(map[string]*lock.RLockedFile),
		},
		nsMutex:       newNSLock(false),
		listPool:      newTreeWalkPool(globalLookupTimeout),
		appendFileMap: make(map[string]*fsAppendFile),
		diskMount:     mountinfo.IsLikelyMountPoint(fsPath),
	}

	// Once the filesystem has initialized hold the read lock for
	// the life time of the server. This is done to ensure that under
	// shared backend mode for FS, remote servers do not migrate
	// or cause changes on backend format.
	fs.fsFormatRlk = rlk

	if !fs.diskMount {
		go fs.diskUsage(globalServiceDoneCh)
	}

	go fs.cleanupStaleMultipartUploads(ctx, globalMultipartCleanupInterval, globalMultipartExpiry, globalServiceDoneCh)

	// Return successfully initialized object layer.
	return fs, nil
}
```

### 获取bucket 列表

`minio/cmd/bucket-handlers.go`

获取bucket 列表，实现分为

1. 从ObjcetAPI 获取具体实现函数
2. 调用函数获取bucket list

```go
//minio/cmd/api-router.go
// ListBuckets 注册路由
apiRouter.Methods("GET").Path("/").HandlerFunc(httpTraceAll(api.ListBucketsHandler))

//minio/cmd/bucket-handlers.go
// ListBucketsHandler - GET Service.
func (api objectAPIHandlers) ListBucketsHandler(w http.ResponseWriter, r *http.Request) {
	ctx := newContext(r, w, "ListBuckets")

	//获取api
	objectAPI := api.ObjectAPI()
	
	//获取bucket list 函数
	listBuckets := objectAPI.ListBuckets

	//从缓存中获取bucket list
	if api.CacheAPI() != nil {
		listBuckets = api.CacheAPI().ListBuckets
	}

	//调用获取bucket 的函数
	var err error
	bucketsInfo, err = listBuckets(ctx)

	//生成响应结果
	response := generateListBucketsResponse(bucketsInfo)
	encodedSuccessResponse := encodeResponse(response)

	//将响应结果返回
	writeSuccessResponseXML(w, encodedSuccessResponse)
}

//minio/cmd/fs-v1.go
// ListBuckets - list all s3 compatible buckets (directories) at fsPath.
func (fs *FSObjects) ListBuckets(ctx context.Context) ([]BucketInfo, error) {
	var bucketInfos []BucketInfo
	entries, err := readDir((fs.fsPath))

	for _, entry := range entries {
		// Ignore all reserved bucket names and invalid bucket names.
		if isReservedOrInvalidBucket(entry) {
			continue
		}

		var fi os.FileInfo
		fi, err = fsStatVolume(ctx, pathJoin(fs.fsPath, entry))
		
		// There seems like no practical reason to check for errors
		// at this point, if there are indeed errors we can simply
		// just ignore such buckets and list only those which
		// return proper Stat information instead.
		if err != nil {
			// Ignore any errors returned here.
			continue
		}

		bucketInfos = append(bucketInfos, BucketInfo{
			Name: fi.Name(),
			// As os.Stat() doesnt carry CreatedTime, use ModTime() as CreatedTime.
			Created: fi.ModTime(),
		})
	}

	// Sort bucket infos by bucket name.
	sort.Sort(byBucketName(bucketInfos))

	// Succes.
	return bucketInfos, nil
}
```

### 获取文件

具体实现包括

1. 注册api 接口
2. 检查请求auth
3. 获取文件信息：需要调用fx-v1 函数处理
4. 获取文件内容：需要调用fx-v1 函数处理
5. 将内容写回请求端
6. 发送获取文件内容通知

从磁盘获取文件内容的实现步骤包括：

1. 检查文件信息
2. 获取文件元信息
3. 打开文件，开始读取文件内容
4. 将文件内容copy 到http 响应中

```go
//minio/cmd/api-router.go
// GetObject
bucket.Methods("GET").Path("/{object:.+}").HandlerFunc(httpTraceHdrs(api.GetObjectHandler))

//minio/cmd/object-handlers.go
// GetObjectHandler 获取文件内容操作
func (api objectAPIHandlers) GetObjectHandler(w http.ResponseWriter, r *http.Request) {
	ctx := newContext(r, w, "GetObject")

	objectAPI := api.ObjectAPI()

	if crypto.S3.IsRequested(r.Header) || crypto.S3KMS.IsRequested(r.Header) { // If SSE-S3 or SSE-KMS present -> AWS fails with undefined error
		writeErrorResponse(w, ErrBadRequest, r.URL)
		return
	}

	vars := mux.Vars(r)
	bucket := vars["bucket"]
	object := vars["object"]

	//获取文件函数
	getObjectInfo := objectAPI.GetObjectInfo
	if api.CacheAPI() != nil {
		getObjectInfo = api.CacheAPI().GetObjectInfo
	}

	//检查请求auth
	if s3Error := checkRequestAuthType(ctx, r, policy.GetObjectAction, bucket, object); s3Error != ErrNone {
		if getRequestAuthType(r) == authTypeAnonymous {
			if globalPolicySys.IsAllowed(policy.Args{
				Action:          policy.ListBucketAction,
				BucketName:      bucket,
				ConditionValues: getConditionValues(r, ""),
				IsOwner:         false,
			}) {
				_, err := getObjectInfo(ctx, bucket, object)
				if toAPIErrorCode(err) == ErrNoSuchKey {
					s3Error = ErrNoSuchKey
				}
			}
		}
		writeErrorResponse(w, s3Error, r.URL)
		return
	}

	//获取文件信息
	objInfo, err := getObjectInfo(ctx, bucket, object)
	
	//如果文件加密，则进行解密
	if objectAPI.IsEncryptionSupported() {
		if apiErr, _ := DecryptObjectInfo(&objInfo, r.Header); apiErr != ErrNone {
			writeErrorResponse(w, apiErr, r.URL)
			return
		}
	}

	//处理range 请求
	var hrange *httpRange

	//解析http 的range 参数
	rangeHeader := r.Header.Get("Range")
	if rangeHeader != "" {
		if hrange, err = parseRequestRange(rangeHeader, objInfo.Size); err != nil {
			if err == errInvalidRange {
				writeErrorResponse(w, ErrInvalidRange, r.URL)
				return
			}
		}
	}

	//校验预置条件
	if checkPreconditions(w, r, objInfo) {
		return
	}

	//获取文件内容
	var startOffset int64
	length := objInfo.Size
	if hrange != nil {
		startOffset = hrange.offsetBegin
		length = hrange.getLength()
	}

	//开始文件读取
	var writer io.Writer
	writer = w

	//如果服务端对文件加密，则需要解密
	if objectAPI.IsEncryptionSupported() {
		s3Encrypted := crypto.S3.IsEncrypted(objInfo.UserDefined)
		if crypto.SSEC.IsRequested(r.Header) || s3Encrypted {
			writer = ioutil.LimitedWriter(writer, startOffset%(64*1024), length)

			writer, startOffset, length, err = DecryptBlocksRequest(writer, r, bucket, object, startOffset, length, objInfo, false)

			if s3Encrypted {
				w.Header().Set(crypto.SSEHeader, crypto.SSEAlgorithmAES256)
			} else {
				w.Header().Set(crypto.SSECAlgorithm, r.Header.Get(crypto.SSECAlgorithm))
				w.Header().Set(crypto.SSECKeyMD5, r.Header.Get(crypto.SSECKeyMD5))
			}
		}
	}

	//设置响应头
	setObjectHeaders(w, objInfo, hrange)
	setHeadGetRespHeaders(w, r.URL.Query())

	//获取文件内容
	getObject := objectAPI.GetObject
	if api.CacheAPI() != nil && !crypto.SSEC.IsRequested(r.Header) && !crypto.S3.IsEncrypted(objInfo.UserDefined) {
		getObject = api.CacheAPI().GetObject
	}

	statusCodeWritten := false
	httpWriter := ioutil.WriteOnClose(writer)

	if hrange != nil && hrange.offsetBegin > -1 {
		statusCodeWritten = true
		w.WriteHeader(http.StatusPartialContent)
	}

	//读取文件内容
	if err = getObject(ctx, bucket, object, startOffset, length, httpWriter, objInfo.ETag); err != nil {
		if !httpWriter.HasWritten() && !statusCodeWritten { // write error response only if no data or headers has been written to client yet
			writeErrorResponse(w, toAPIErrorCode(err), r.URL)
		}
		httpWriter.Close()
		return
	}

	//将内容写回请求端
	if err = httpWriter.Close(); err != nil {
		if !httpWriter.HasWritten() && !statusCodeWritten { // write error response only if no data or headers has been written to client yet
			writeErrorResponse(w, toAPIErrorCode(err), r.URL)
			return
		}
	}

	//获取请求者信息
	host, port, err := net.SplitHostPort(handlers.GetSourceIP(r))
	if err != nil {
		host, port = "", ""
	}

	//发送获取文件内容通知
	sendEvent(eventArgs{
		EventName:  event.ObjectAccessedGet,
		BucketName: bucket,
		Object:     objInfo,
		ReqParams:  extractReqParams(r),
		UserAgent:  r.UserAgent(),
		Host:       host,
		Port:       port,
	})
}

//fs-v1.go
//objectAPI.GetObjectInfo 获取文件元信息
func (fs *FSObjects) GetObjectInfo(ctx context.Context, bucket, object string) (oi ObjectInfo, e error) {
	oi, err := fs.getObjectInfoWithLock(ctx, bucket, object)
	if err == errCorruptedFormat || err == io.EOF {
		objectLock := fs.nsMutex.NewNSLock(bucket, object)
		if err = objectLock.GetLock(globalObjectTimeout); err != nil {
			return oi, toObjectErr(err, bucket, object)
		}

		fsMetaPath := pathJoin(fs.fsPath, minioMetaBucket, bucketMetaPrefix, bucket, object, fs.metaJSONFile)
		err = fs.createFsJSON(object, fsMetaPath)
		objectLock.Unlock()
		if err != nil {
			return oi, toObjectErr(err, bucket, object)
		}

		oi, err = fs.getObjectInfoWithLock(ctx, bucket, object)
	}
	return oi, toObjectErr(err, bucket, object)
}

//minio/cmd/fs-v1.go
// GetObject 从磁盘读取文件内容
func (fs *FSObjects) GetObject(ctx context.Context, bucket, object string, offset int64, length int64, writer io.Writer, etag string) (err error) {
	if err = checkGetObjArgs(ctx, bucket, object); err != nil {
		return err
	}

	// Lock the object before reading.
	objectLock := fs.nsMutex.NewNSLock(bucket, object)
	if err := objectLock.GetRLock(globalObjectTimeout); err != nil {
		logger.LogIf(ctx, err)
		return err
	}
	defer objectLock.RUnlock()
	return fs.getObject(ctx, bucket, object, offset, length, writer, etag, true)
}

//minio/cmd/fs-v1.go
// getObject - wrapper for GetObject
func (fs *FSObjects) getObject(ctx context.Context, bucket, object string, offset int64, length int64, writer io.Writer, etag string, lock bool) (err error) {
	//检查文件信息
	if _, err = fs.statBucketDir(ctx, bucket); err != nil {
		return toObjectErr(err, bucket)
	}

	// Offset cannot be negative.
	if offset < 0 {
		logger.LogIf(ctx, errUnexpected)
		return toObjectErr(errUnexpected, bucket, object)
	}

	// Writer cannot be nil.
	if writer == nil {
		logger.LogIf(ctx, errUnexpected)
		return toObjectErr(errUnexpected, bucket, object)
	}

	// If its a directory request, we return an empty body.
	if hasSuffix(object, slashSeparator) {
		_, err = writer.Write([]byte(""))
		logger.LogIf(ctx, err)
		return toObjectErr(err, bucket, object)
	}

	//获取文件元信息
	if bucket != minioMetaBucket {
		fsMetaPath := pathJoin(fs.fsPath, minioMetaBucket, bucketMetaPrefix, bucket, object, fs.metaJSONFile)
		if lock {
			_, err = fs.rwPool.Open(fsMetaPath)
			if err != nil && err != errFileNotFound {
				logger.LogIf(ctx, err)
				return toObjectErr(err, bucket, object)
			}
			defer fs.rwPool.Close(fsMetaPath)
		}
	}

	//获取文件etag
	if etag != "" && etag != defaultEtag {
		objEtag, perr := fs.getObjectETag(ctx, bucket, object, lock)
		if perr != nil {
			return toObjectErr(perr, bucket, object)
		}
		if objEtag != etag {
			logger.LogIf(ctx, InvalidETag{})
			return toObjectErr(InvalidETag{}, bucket, object)
		}
	}

	//打开文件，开始读取文件内容
	fsObjPath := pathJoin(fs.fsPath, bucket, object)
	reader, size, err := fsOpenFile(ctx, fsObjPath, offset)
	defer reader.Close()

	bufSize := int64(readSizeV1)
	if length > 0 && bufSize > length {
		bufSize = length
	}

	// For negative length we read everything.
	if length < 0 {
		length = size - offset
	}

	// Reply back invalid range if the input offset and length fall out of range.
	if offset > size || offset+length > size {
		err = InvalidRange{offset, length, size}
		logger.LogIf(ctx, err)
		return err
	}

	// Allocate a staging buffer.
	buf := make([]byte, int(bufSize))

	//将文件内容copy 到http 响应中
	_, err = io.CopyBuffer(writer, io.LimitReader(reader, length), buf)
	logger.LogIf(ctx, err)
	return toObjectErr(err, bucket, object)
}
```

### 处理上传

object-handlers.go 函数 PutObjectHandler 实现

实现步骤分为：

1. 注册api 接口
2. 检查上传参数字段
3. 获取元信息，上传auth 验证
4. 创建hashReader 计算文件内容hash 值，检查是否开启了服务端文件加密
5. 执行pubObject 创建对象
6. 从输入流中读取内容写入文件，并且增加fs.json 用于保存元信息
7. 对object 加锁避免同时写入，调用 fs.putObject 进行文件保存处理
8. 处理文件元信息，检查上传文件参数
9. 上传的文件会先写入临时文件，再将临时文件重命名为正式文件
10. 返回文件元信息
11. 发送文件上传成功事件通知

```go
//minio/cmd/api-router.go
//PutObject api 接口定义
bucket.Methods("PUT").Path("/{object:.+}").HandlerFunc(httpTraceHdrs(api.PutObjectHandler))

//minio/cmd/object-handlers.go
//PutObjectHandler 实现文件上传操作
func (api objectAPIHandlers) PutObjectHandler(w http.ResponseWriter, r *http.Request) {
	ctx := newContext(r, w, "PutObject")

	objectAPI := api.ObjectAPI()

	if !objectAPI.IsEncryptionSupported() && crypto.S3KMS.IsRequested(r.Header) {
		writeErrorResponse(w, ErrNotImplemented, r.URL) // SSE-KMS is not supported
		return
	}

	//获取参数
	vars := mux.Vars(r)
	bucket := vars["bucket"]
	object := vars["object"]

	//检验客户端传送的md5 字段
	md5Bytes, err := checkValidMD5(r.Header)

	//检查 Content-Length 字段
	size := r.ContentLength
	rAuthType := getRequestAuthType(r)
	if rAuthType == authTypeStreamingSigned {
		if sizeStr, ok := r.Header["X-Amz-Decoded-Content-Length"]; ok {
			if sizeStr[0] == "" {
				writeErrorResponse(w, ErrMissingContentLength, r.URL)
				return
			}
			size, err = strconv.ParseInt(sizeStr[0], 10, 64)
			if err != nil {
				writeErrorResponse(w, toAPIErrorCode(err), r.URL)
				return
			}
		}
	}

	//检查是否达到单次上传的最大长度
	if isMaxObjectSize(size) {
		writeErrorResponse(w, ErrEntityTooLarge, r.URL)
		return
	}

	//获取元数据
	metadata, err := extractMetadata(ctx, r)

	//获取上传的content-encoding 字段
	if rAuthType == authTypeStreamingSigned {
		if contentEncoding, ok := metadata["content-encoding"]; ok {
			contentEncoding = trimAwsChunkedContentEncoding(contentEncoding)
			if contentEncoding != "" {
				metadata["content-encoding"] = contentEncoding
			} else {
				delete(metadata, "content-encoding")
			}
		}
	}

	var (
		md5hex    = hex.EncodeToString(md5Bytes)
		sha256hex = ""
		reader    io.Reader
		s3Err     APIErrorCode
		putObject = objectAPI.PutObject //objectAPI 操作层
	)

	reader = r.Body

	//上传auth 验证
	switch rAuthType { //根据不同的验证类型，进行不同检测处理
	default:
		// For all unknown auth types return error.
		writeErrorResponse(w, ErrAccessDenied, r.URL)
		return
	case authTypeAnonymous:
		if !globalPolicySys.IsAllowed(policy.Args{
			Action:          policy.PutObjectAction,
			BucketName:      bucket,
			ConditionValues: getConditionValues(r, ""),
			IsOwner:         false,
			ObjectName:      object,
		}) {
			writeErrorResponse(w, ErrAccessDenied, r.URL)
			return
		}
	case authTypeStreamingSigned:
		// Initialize stream signature verifier.
		reader, s3Err = newSignV4ChunkedReader(r)
		if s3Err != ErrNone {
			writeErrorResponse(w, s3Err, r.URL)
			return
		}
	case authTypeSignedV2, authTypePresignedV2:
		s3Err = isReqAuthenticatedV2(r)
		if s3Err != ErrNone {
			writeErrorResponse(w, s3Err, r.URL)
			return
		}

	case authTypePresigned, authTypeSigned:
		if s3Err = reqSignatureV4Verify(r, globalServerConfig.GetRegion()); s3Err != ErrNone {
			writeErrorResponse(w, s3Err, r.URL)
			return
		}
		if !skipContentSha256Cksum(r) {
			sha256hex = getContentSha256Cksum(r)
		}
	}

	//创建hashReader，计算文件内容hash 值
	hashReader, err := hash.NewReader(reader, size, md5hex, sha256hex)
	
	// Deny if WORM is enabled
	if globalWORMEnabled {
		if _, err = objectAPI.GetObjectInfo(ctx, bucket, object); err == nil {
			writeErrorResponse(w, ErrMethodNotAllowed, r.URL)
			return
		}
	}

	//检查是否开启了服务端文件加密
	if objectAPI.IsEncryptionSupported() {
		if hasServerSideEncryptionHeader(r.Header) && !hasSuffix(object, slashSeparator) { // handle SSE requests
			reader, err = EncryptRequest(hashReader, r, bucket, object, metadata)
			
			info := ObjectInfo{Size: size}
			hashReader, err = hash.NewReader(reader, info.EncryptedSize(), "", "") // do not try to verify encrypted content
		}
	}

	if api.CacheAPI() != nil && !hasServerSideEncryptionHeader(r.Header) {
		putObject = api.CacheAPI().PutObject
	}

	//执行pubObject 创建对象
	objInfo, err := putObject(ctx, bucket, object, hashReader, metadata)

	//设置响应头，返回文件ETag
	w.Header().Set("ETag", "\""+objInfo.ETag+"\"")
	if objectAPI.IsEncryptionSupported() {
		if crypto.S3.IsEncrypted(objInfo.UserDefined) {
			w.Header().Set(crypto.SSEHeader, crypto.SSEAlgorithmAES256)
		}
		if crypto.SSEC.IsRequested(r.Header) {
			w.Header().Set(crypto.SSECAlgorithm, r.Header.Get(crypto.SSECAlgorithm))
			w.Header().Set(crypto.SSECKeyMD5, r.Header.Get(crypto.SSECKeyMD5))
		}
	}

	//响应处理成功数据
	writeSuccessResponseHeadersOnly(w)

	// Get host and port from Request.RemoteAddr.
	host, port, err := net.SplitHostPort(handlers.GetSourceIP(r))
	if err != nil {
		host, port = "", ""
	}

	//发送文件上传成功事件通知
	sendEvent(eventArgs{
		EventName:  event.ObjectCreatedPut,
		BucketName: bucket,
		Object:     objInfo,
		ReqParams:  extractReqParams(r),
		UserAgent:  r.UserAgent(),
		Host:       host,
		Port:       port,
	})
}

//minio/cmd/fs-v1.go
// PutObject 从输入流中读取内容写入文件，并且增加fs.json 用于保存云信息
func (fs *FSObjects) PutObject(ctx context.Context, bucket string, object string, data *hash.Reader, metadata map[string]string) (objInfo ObjectInfo, retErr error) {
	if err := checkPutObjectArgs(ctx, bucket, object, fs, data.Size()); err != nil {
		return ObjectInfo{}, err
	}
	// Lock the object.
	objectLock := fs.nsMutex.NewNSLock(bucket, object)
	if err := objectLock.GetLock(globalObjectTimeout); err != nil {
		logger.LogIf(ctx, err)
		return objInfo, err
	}
	defer objectLock.Unlock()
	return fs.putObject(ctx, bucket, object, data, metadata)
}

//minio/cmd/fs-v1.go
// putObject 实现将文件写入磁盘的操作
func (fs *FSObjects) putObject(ctx context.Context, bucket string, object string, data *hash.Reader, metadata map[string]string) (objInfo ObjectInfo, retErr error) {
	// No metadata is set, allocate a new one.
	meta := make(map[string]string)
	for k, v := range metadata {
		meta[k] = v
	}
	
	//文件云信息
	fsMeta := newFSMetaV1()
	fsMeta.Meta = meta

	//创建文件目录处理，只创建目录
	if isObjectDir(object, data.Size()) {
		// Check if an object is present as one of the parent dir.
		if fs.parentDirIsObject(ctx, bucket, path.Dir(object)) {
			logger.LogIf(ctx, errFileAccessDenied)
			return ObjectInfo{}, toObjectErr(errFileAccessDenied, bucket, object)
		}
		if err = mkdirAll(pathJoin(fs.fsPath, bucket, object), 0777); err != nil {
			logger.LogIf(ctx, err)
			return ObjectInfo{}, toObjectErr(err, bucket, object)
		}
		var fi os.FileInfo
		if fi, err = fsStatDir(ctx, pathJoin(fs.fsPath, bucket, object)); err != nil {
			return ObjectInfo{}, toObjectErr(err, bucket, object)
		}
		return fsMeta.ToObjectInfo(bucket, object, fi), nil
	}

	//检查上传文件参数
	if err = checkPutObjectArgs(ctx, bucket, object, fs, data.Size()); err != nil {
		return ObjectInfo{}, err
	}

	//检查上传文件的部分路径是不是一个object
	if fs.parentDirIsObject(ctx, bucket, path.Dir(object)) {
		logger.LogIf(ctx, errFileAccessDenied)
		return ObjectInfo{}, toObjectErr(errFileAccessDenied, bucket, object)
	}

	//检查文件大小
	if data.Size() < 0 {
		logger.LogIf(ctx, errInvalidArgument)
		return ObjectInfo{}, errInvalidArgument
	}

	//元信息写入处理
	var wlk *lock.LockedFile
	if bucket != minioMetaBucket {
		bucketMetaDir := pathJoin(fs.fsPath, minioMetaBucket, bucketMetaPrefix)

		fsMetaPath := pathJoin(bucketMetaDir, bucket, object, fs.metaJSONFile)
		wlk, err = fs.rwPool.Create(fsMetaPath)
		
		// This close will allow for locks to be synchronized on `fs.json`.
		defer wlk.Close()
		defer func() {
			// Remove meta file when PutObject encounters any error
			if retErr != nil {
				tmpDir := pathJoin(fs.fsPath, minioMetaTmpBucket, fs.fsUUID)
				fsRemoveMeta(ctx, bucketMetaDir, fsMetaPath, tmpDir)
			}
		}()
	}

	//上传的文件会先写入临时文件，最后将临时文件重命名为正式文件，因为文件最开始写入临时文件，所以在服务出现异常时可以很方便的清理掉
	tempObj := mustGetUUID()

	//创建一个buffer
	bufSize := int64(readSizeV1)
	if size := data.Size(); size > 0 && bufSize > size {
		bufSize = size
	}
	buf := make([]byte, int(bufSize))

	//将上传写入临时文件
	fsTmpObjPath := pathJoin(fs.fsPath, minioMetaTmpBucket, fs.fsUUID, tempObj)
	bytesWritten, err := fsCreateFile(ctx, fsTmpObjPath, data, buf, data.Size())
	if err != nil {
		fsRemoveFile(ctx, fsTmpObjPath)
		return ObjectInfo{}, toObjectErr(err, bucket, object)
	}

	//获取文件md5 值
	fsMeta.Meta["etag"] = hex.EncodeToString(data.MD5Current())

	//在处理完成后删除临时文件
	defer fsRemoveFile(ctx, fsTmpObjPath)

	//写入临时文件后，在重命名为正式文件
	fsNSObjPath := pathJoin(fs.fsPath, bucket, object)

	if err = fsRenameFile(ctx, fsTmpObjPath, fsNSObjPath); err != nil {
		return ObjectInfo{}, toObjectErr(err, bucket, object)
	}

	//写入文件元信息
	if bucket != minioMetaBucket {
		// Write FS metadata after a successful namespace operation.
		if _, err = fsMeta.WriteTo(wlk); err != nil {
			return ObjectInfo{}, toObjectErr(err, bucket, object)
		}
	}

	//获取文件信息
	fi, err := fsStatFile(ctx, pathJoin(fs.fsPath, bucket, object))
	if err != nil {
		return ObjectInfo{}, toObjectErr(err, bucket, object)
	}

	//返回文件元信息
	return fsMeta.ToObjectInfo(bucket, object, fi), nil
}
```


## 参考

- [https://github.com/minio/minio/blob/master/main.go](https://github.com/minio/minio/blob/master/main.go)

