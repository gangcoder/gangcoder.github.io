<!-- ---
title: jaeger hotrod
date: 2019-08-15 15:20:49
category: showcode, jaeger
--- -->

# jaeger hotrod

> jaeger hotrod 是jaeger 示例程序，产生调用链数据，用于学习和测试jaeger。

## 入口

开启hotrod 命令：

```
hotrod all
```

```go
// allCmd represents the all command
var allCmd = &cobra.Command{
	Use:   "all",
	Short: "Starts all services",
	Long:  `Starts all services.`,
	RunE: func(cmd *cobra.Command, args []string) error {
		logger.Info("Starting all services")
		go customerCmd.RunE(customerCmd, args)
		go driverCmd.RunE(driverCmd, args)
		go routeCmd.RunE(routeCmd, args)
		return frontendCmd.RunE(frontendCmd, args)
	},
}
```

hotrod 模块：

1. 前端入口 frontend
2. 服务端 customerCmd
3. 服务端 driverCmd
4. 服务端 routeCmd

前端入口调用各服务端产生调用链，上报到 jaeger。

## 前端入口模块

```go
// github.com/jaegertracing/jaeger/examples/hotrod/cmd/frontend.go

// 创建服务
server := frontend.NewServer(
	options,
	tracing.Init("frontend", metricsFactory, logger),
	logger,
)

// 运行服务
server.Run()
```

### 创建服务

```go
// github.com/jaegertracing/jaeger/examples/hotrod/pkg/tracing/init.go
// 创建一条trace
tracing.Init("frontend", metricsFactory, logger),

// 创建trace 实例
func Init(serviceName string, metricsFactory metrics.Factory, logger log.Factory) opentracing.Tracer {
	// ...
	cfg.ServiceName = serviceName
	cfg.Sampler.Type = "const"
	cfg.Sampler.Param = 1

	// ...
	tracer, _, err := cfg.NewTracer(
		config.Logger(jaegerLogger),
		config.Metrics(metricsFactory),
		config.Observer(rpcmetrics.NewObserver(metricsFactory, rpcmetrics.DefaultNameNormalizer)),
	)
	return tracer
}


// NewServer 创建frontend 服务
func NewServer(options ConfigOptions, tracer opentracing.Tracer, logger log.Factory) *Server {
	assetFS := FS(false)
	return &Server{
		hostPort: options.FrontendHostPort,
		tracer:   tracer,
		logger:   logger,
		bestETA:  newBestETA(tracer, logger, options),
		assetFS:  assetFS,
	}
}
```

### 运行服务

```go
// github.com/jaegertracing/jaeger/examples/hotrod/services/frontend/server.go
// Run starts the frontend server
func (s *Server) Run() error {
	mux := s.createServeMux()
	// ...
	return http.ListenAndServe(s.hostPort, mux)
}

// 注册路由和处理函数
func (s *Server) createServeMux() http.Handler {
	mux := tracing.NewServeMux(s.tracer)
	mux.Handle("/", http.FileServer(s.assetFS))
	mux.Handle("/dispatch", http.HandlerFunc(s.dispatch))
	return mux
}

// 路由处理handler
func (s *Server) dispatch(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	
	// ...
	customerID := r.Form.Get("customer")
	if customerID == "" {
		http.Error(w, "Missing required 'customer' parameter", http.StatusBadRequest)
		return
	}

	// 请求数据，这里会产生span 数据
	response, err := s.bestETA.Get(ctx, customerID)
	
	// ...
	data, err := json.Marshal(response)
	
	w.Header().Set("Content-Type", "application/json")
	w.Write(data)
}
```

### 请求查询处理

```go
// github.com/jaegertracing/jaeger/examples/hotrod/services/frontend/best_eta.go
// 创建请求client
func newBestETA(tracer opentracing.Tracer, logger log.Factory, options ConfigOptions) *bestETA {
	return &bestETA{
		customer: customer.NewClient(
			tracer,
			logger.With(zap.String("component", "customer_client")),
			options.CustomerHostPort,
		),
		driver: driver.NewClient(
			tracer,
			logger.With(zap.String("component", "driver_client")),
			options.DriverHostPort,
		),
		route: route.NewClient(
			tracer,
			logger.With(zap.String("component", "route_client")),
			options.RouteHostPort,
		),
		pool:   pool.New(config.RouteWorkerPoolSize),
		logger: logger,
	}
}

// github.com/jaegertracing/jaeger/examples/hotrod/services/frontend/best_eta.go
func (eta *bestETA) Get(ctx context.Context, customerID string) (*Response, error) {
	// ...
	// 查询customer 数据
	customer, err := eta.customer.Get(ctx, customerID)

	if span := opentracing.SpanFromContext(ctx); span != nil {
		span.SetBaggageItem("customer", customer.Name)
	}

	// 查询drivers 数据
	drivers, err := eta.driver.FindNearest(ctx, customer.Location)

	// 获取路由数据
	results := eta.getRoutes(ctx, customer, drivers)
	
	// 结果数据处理
	resp := &Response{ETA: math.MaxInt64}
	for _, result := range results {
		if result.err != nil {
			return nil, err
		}
		if result.route.ETA < resp.ETA {
			resp.ETA = result.route.ETA
			resp.Driver = result.driver
		}
	}
	// ...

	return resp, nil
}
```

### Custome 客户端请求实现

```go
// github.com/jaegertracing/jaeger/examples/hotrod/services/customer/client.go
// Get implements customer.Interface#Get as an RPC
func (c *Client) Get(ctx context.Context, customerID string) (*Customer, error) {
	// ...
	url := fmt.Sprintf("http://" + c.hostPort + "/customer?customer=%s", customerID)
	fmt.Println(url)
	var customer Customer
	if err := c.client.GetJSON(ctx, "/customer", url, &customer); err != nil {
		return nil, err
	}
	return &customer, nil
}

// GetJSON 记录请求耗时
func (c *HTTPClient) GetJSON(ctx context.Context, endpoint string, url string, out interface{}) error {
	req, err := http.NewRequest("GET", url, nil)
	
	// ...
	req = req.WithContext(ctx)
	req, ht := nethttp.TraceRequest(c.Tracer, req, nethttp.OperationName("HTTP GET: "+endpoint))
	defer ht.Finish()

	res, err := c.Client.Do(req)
	// ...
	defer res.Body.Close()
	// ...
	decoder := json.NewDecoder(res.Body)
	return decoder.Decode(out)
}
```

### Route 客户端请求实现

```go
// github.com/jaegertracing/jaeger/examples/hotrod/services/route/client.go
// NewClient creates a new route.Client
func NewClient(tracer opentracing.Tracer, logger log.Factory, hostPort string) *Client {
	return &Client{
		tracer: tracer,
		logger: logger,
		client: &tracing.HTTPClient{
			Client: &http.Client{Transport: &nethttp.Transport{}},
			Tracer: tracer,
		},
		hostPort: hostPort,
	}
}

// FindRoute 请求调用并且记录调用span 数据
func (c *Client) FindRoute(ctx context.Context, pickup, dropoff string) (*Route, error) {
	// ...
	v := url.Values{}
	v.Set("pickup", pickup)
	v.Set("dropoff", dropoff)
	url := "http://" + c.hostPort + "/route?" + v.Encode()
	var route Route
	if err := c.client.GetJSON(ctx, "/route", url, &route); err != nil {
		c.logger.For(ctx).Error("Error getting route", zap.Error(err))
		return nil, err
	}
	return &route, nil
}
```

## 服务端 customerCmd

```go
server := customer.NewServer(
	net.JoinHostPort("0.0.0.0", strconv.Itoa(customerPort)),
	tracing.Init("customer", metricsFactory, logger),
	metricsFactory,
	logger,
)
return logError(zapLogger, server.Run())

jaeger/examples/hotrod/services/customer/server.go
// NewServer 创建customer 服务
func NewServer(hostPort string, tracer opentracing.Tracer, metricsFactory metrics.Factory, logger log.Factory) *Server {
	return &Server{
		hostPort: hostPort,
		tracer:   tracer,
		logger:   logger,
		database: newDatabase(
			tracing.Init("mysql", metricsFactory, logger),
			logger.With(zap.String("component", "mysql")),
		),
	}
}

// Run 运行服务
func (s *Server) Run() error {
	mux := s.createServeMux()
	// ...
	return http.ListenAndServe(s.hostPort, mux)
}

// 注册路由handler
func (s *Server) createServeMux() http.Handler {
	mux := tracing.NewServeMux(s.tracer)
	mux.Handle("/customer", http.HandlerFunc(s.customer))
	return mux
}

func (s *Server) customer(w http.ResponseWriter, r *http.Request) {
	// ...
	customerID := r.Form.Get("customer")
	
	response, err := s.database.Get(ctx, customerID)
	
	data, err := json.Marshal(response)

	// ...
	w.Header().Set("Content-Type", "application/json")
	w.Write(data)
}
```


## 服务端 driverCmd

```go
server := driver.NewServer(
	net.JoinHostPort("0.0.0.0", strconv.Itoa(driverPort)),
	tracing.Init("driver", metricsFactory, logger),
	metricsFactory,
	logger,
)
return logError(zapLogger, server.Run())


// NewServer 创建driver 服务
func NewServer(hostPort string, tracer opentracing.Tracer, metricsFactory metrics.Factory, logger log.Factory) *Server {
	// ...
	server := thrift.NewServer(ch)

	return &Server{
		hostPort: hostPort,
		tracer:   tracer,
		logger:   logger,
		ch:       ch,
		server:   server,
		redis:    newRedis(metricsFactory, logger),
	}
}

// Run 运行服务，注册处理函数
func (s *Server) Run() error {
	s.server.Register(driver.NewTChanDriverServer(s))
	if err := s.ch.ListenAndServe(s.hostPort); err != nil {
		s.logger.Bg().Fatal("Unable to start tchannel server", zap.Error(err))
	}
	// ...
	select {}
}

// FindNearest 实现接口
func (s *Server) FindNearest(ctx thrift.Context, location string) ([]*driver.DriverLocation, error) {
	// 查询数据
	driverIDs := s.redis.FindDriverIDs(ctx, location)
	// ...
	return retMe, nil
}

// FindDriverIDs 创建span 数据
func (r *Redis) FindDriverIDs(ctx context.Context, location string) []string {
	if span := opentracing.SpanFromContext(ctx); span != nil {
		span := r.tracer.StartSpan("FindDriverIDs", opentracing.ChildOf(span.Context()))
		span.SetTag("param.location", location)
		ext.SpanKindRPCClient.Set(span)
		defer span.Finish()
		ctx = opentracing.ContextWithSpan(ctx, span)
	}
	// simulate RPC delay
	delay.Sleep(config.RedisFindDelay, config.RedisFindDelayStdDev)

	drivers := make([]string, 10)
	// ...
	return drivers
}
```

## 服务端 routeCmd


```go
server := route.NewServer(
	net.JoinHostPort("0.0.0.0", strconv.Itoa(routePort)),
	tracing.Init("route", metricsFactory, logger),
	logger,
)
return logError(zapLogger, server.Run())

// github.com/jaegertracing/jaeger/examples/hotrod/services/route/server.go
// NewServer 创建路由服务
func NewServer(hostPort string, tracer opentracing.Tracer, logger log.Factory) *Server {
	return &Server{
		hostPort: hostPort,
		tracer:   tracer,
		logger:   logger,
	}
}

// Run starts the Route server
func (s *Server) Run() error {
	mux := s.createServeMux()
	// ...
	return http.ListenAndServe(s.hostPort, mux)
}

// 注册路由handler
func (s *Server) createServeMux() http.Handler {
	mux := tracing.NewServeMux(s.tracer)
	mux.Handle("/route", http.HandlerFunc(s.route))
	mux.Handle("/debug/vars", expvar.Handler()) // expvar
	mux.Handle("/metrics", promhttp.Handler())  // Prometheus
	return mux
}

func (s *Server) route(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	
	// ...
	pickup := r.Form.Get("pickup")
	
	// ...
	response := computeRoute(ctx, pickup, dropoff)

	data, err := json.Marshal(response)

	w.Header().Set("Content-Type", "application/json")
	w.Write(data)
}
```

## 参考资料

- github.com/jaegertracing/jaeger > examples/hotrod/main.go

