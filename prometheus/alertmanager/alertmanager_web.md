<!-- ---
title: 告警Web 实现
date: 2020-03-01 14:51:33
category: showcode, prometheus, alertmanager
--- -->

# 告警Web 实现

告警中心Web 功能实现，提供告警中心管理web 页面模板和API 接口。

1. 创建API 端点控制器
2. 更新API 配置
3. 注册UI 路由
4. 注册API 路由

![](images/alertmanager_web.svg)


```go
// 创建API 端点控制器
api, err := api.New(...)

// 更新API 配置
api.Update(conf, func(labels model.LabelSet) {
    inhibitor.Mutes(labels)
    silencer.Mutes(labels)
})

// 注册UI 路由
ui.Register(router, webReload, logger)
// 注册API 路由
mux := api.Register(router, *routePrefix)
```

## 1. 创建API 端点控制器

```go
// API 告警中心的web api 接口
type API struct {
	v1                       *apiv1.API
	v2                       *apiv2.API // v2 版本遵循swagger 规范
	requestsInFlight         prometheus.Gauge
	concurrencyLimitExceeded prometheus.Counter
	timeout                  time.Duration
	inFlightSem              chan struct{}
}


// API v1 api
type API struct {
	alerts   provider.Alerts
	silences *silence.Silences
	config   *config.Config
	route    *dispatch.Route
}

// API v2 api
type API struct {
	peer           *cluster.Peer
	silences       *silence.Silences
	alerts         provider.Alerts
	alertGroups    groupsFn
	getAlertStatus getAlertStatusFn
	uptime         time.Time
	alertmanagerConfig *config.Config
	route              *dispatch.Route
	setAlertStatus     setAlertStatusFn
}

// 创建api 端点控制器实例
api, err := api.New(api.Options{
    Alerts:      alerts,
    Silences:    silences,
    StatusFunc:  marker.Status,
    Peer:        peer,
    Timeout:     *httpTimeout,
    Concurrency: *getConcurrency,
    Logger:      log.With(logger, "component", "api"),
    Registry:    prometheus.DefaultRegisterer,
    GroupFunc:   groupFn,
})

// github.com/prometheus/alertmanager/api/api.go
func New(opts Options) (*API, error) {
	v1 := apiv1.New(
		opts.Alerts,
		opts.Silences,
		opts.StatusFunc,
		opts.Peer,
		log.With(l, "version", "v1"),
		opts.Registry,
	)

	v2, err := apiv2.NewAPI(
		opts.Alerts,
		opts.GroupFunc,
		opts.StatusFunc,
		opts.Silences,
		opts.Peer,
		log.With(l, "version", "v2"),
		opts.Registry,
	)
    // ...
	return &API{
		v1:                       v1,
		v2:                       v2,
		requestsInFlight:         requestsInFlight,
		concurrencyLimitExceeded: concurrencyLimitExceeded,
		timeout:                  opts.Timeout,
		inFlightSem:              make(chan struct{}, concurrency),
	}, nil
}

// github.com/prometheus/alertmanager/api/v1/api.go
func New(
	alerts provider.Alerts,
	silences *silence.Silences,
	sf getAlertStatusFn,
	peer *cluster.Peer,
	l log.Logger,
	r prometheus.Registerer,
) *API {
    // ...
	return &API{
		alerts:         alerts,
		silences:       silences,
		getAlertStatus: sf,
		uptime:         time.Now(),
		peer:           peer,
		logger:         l,
		m:              metrics.NewAlerts("v1", r),
	}
}

// github.com/prometheus/alertmanager/api/v2/api.go
func NewAPI(
	alerts provider.Alerts,
	gf groupsFn,
	sf getAlertStatusFn,
	silences *silence.Silences,
	peer *cluster.Peer,
	l log.Logger,
	r prometheus.Registerer,
) (*API, error) {
	api := API{
		alerts:         alerts,
		getAlertStatus: sf,
		alertGroups:    gf,
		peer:           peer,
		silences:       silences,
		logger:         l,
		m:              metrics.NewAlerts("v2", r),
		uptime:         time.Now(),
	}

	// 加载 swagger 配置文件
    
	// create new service API
	openAPI := operations.NewAlertmanagerAPI(swaggerSpec)
    // 中间件
	openAPI.Middleware = func(b middleware.Builder) http.Handler {
		return middleware.Spec("", swaggerSpec.Raw(), openAPI.Context().RoutesHandler(b))
	}

    // 注册路由
	openAPI.AlertGetAlertsHandler = alert_ops.GetAlertsHandlerFunc(api.getAlertsHandler)
	openAPI.AlertPostAlertsHandler = alert_ops.PostAlertsHandlerFunc(api.postAlertsHandler)
	openAPI.AlertgroupGetAlertGroupsHandler = alertgroup_ops.GetAlertGroupsHandlerFunc(api.getAlertGroupsHandler)
	openAPI.GeneralGetStatusHandler = general_ops.GetStatusHandlerFunc(api.getStatusHandler)
	openAPI.ReceiverGetReceiversHandler = receiver_ops.GetReceiversHandlerFunc(api.getReceiversHandler)
	openAPI.SilenceDeleteSilenceHandler = silence_ops.DeleteSilenceHandlerFunc(api.deleteSilenceHandler)
	openAPI.SilenceGetSilenceHandler = silence_ops.GetSilenceHandlerFunc(api.getSilenceHandler)
	openAPI.SilenceGetSilencesHandler = silence_ops.GetSilencesHandlerFunc(api.getSilencesHandler)
	openAPI.SilencePostSilencesHandler = silence_ops.PostSilencesHandlerFunc(api.postSilencesHandler)

	openAPI.Logger = func(s string, i ...interface{}) { level.Error(api.logger).Log(i...) }

    // 开启服务handler
	handleCORS := cors.Default().Handler
	api.Handler = handleCORS(openAPI.Serve(nil))

	return &api, nil
}
```

## 2. 更新API 配置

```go
// 更新配置
api.Update(conf, func(labels model.LabelSet) {
    inhibitor.Mutes(labels)
    silencer.Mutes(labels)
})

// github.com/prometheus/alertmanager/api/api.go
func (api *API) Update(cfg *config.Config, setAlertStatus func(model.LabelSet)) {
	api.v1.Update(cfg)
	api.v2.Update(cfg, setAlertStatus)
}

// Update v1
func (api *API) Update(cfg *config.Config) {
    // ...
	api.config = cfg
	api.route = dispatch.NewRoute(cfg.Route, nil)
}

// Update v2
func (api *API) Update(cfg *config.Config, setAlertStatus setAlertStatusFn) {
	// ...
	api.alertmanagerConfig = cfg
	api.route = dispatch.NewRoute(cfg.Route, nil)
	api.setAlertStatus = setAlertStatus
}
```

## 3. 注册UI 路由

```go
// 注册ui 路由
ui.Register(router, webReload, logger)

// Register 注册静态文件模板
func Register(r *route.Router, reloadCh chan<- chan error, logger log.Logger) {
	r.Get("/metrics", promhttp.Handler().ServeHTTP)

    // 静态文件
	r.Get("/", func(w http.ResponseWriter, req *http.Request) {
		disableCaching(w)

		req.URL.Path = "/static/"
		fs := http.FileServer(asset.Assets)
		fs.ServeHTTP(w, req)
	})
    // ...
}
```

## 4. 注册API 路由

```go
// 注册api 路由
mux := api.Register(router, *routePrefix)

// github.com/prometheus/alertmanager/api/api.go
func (api *API) Register(r *route.Router, routePrefix string) *http.ServeMux {
    // 注册v1 接口
	api.v1.Register(r.WithPrefix("/api/v1"))

	mux := http.NewServeMux()
	mux.Handle("/", api.limitHandler(r))
    // ...
    // 注册v2 接口
	mux.Handle(
		apiPrefix+"/api/v2/",
		api.limitHandler(http.StripPrefix(apiPrefix+"/api/v2", api.v2.Handler)),
	)

	return mux
}

// v1 端口注册
func (api *API) Register(r *route.Router) {
    // ...
	r.Options("/*path", wrap(func(w http.ResponseWriter, r *http.Request) {}))

	r.Get("/status", wrap(api.status))
	r.Get("/receivers", wrap(api.receivers))

	r.Get("/alerts", wrap(api.listAlerts))
	r.Post("/alerts", wrap(api.addAlerts))

	r.Get("/silences", wrap(api.listSilences))
	r.Post("/silences", wrap(api.setSilence))
	r.Get("/silence/:sid", wrap(api.getSilence))
	r.Del("/silence/:sid", wrap(api.delSilence))
}

// 告警接收接口
func (api *API) addAlerts(w http.ResponseWriter, r *http.Request) {
    // 读取告警
    // ...
    api.receive(r, &alerts)
    
    // 保存告警
	api.insertAlerts(w, r, alerts...)
}

func (api *API) insertAlerts(w http.ResponseWriter, r *http.Request, alerts ...*types.Alert) {
    // ...
    // 检查告警有效性
	for _, a := range alerts {
		// ...
		if err := a.Validate(); err != nil {
			validationErrs.Add(err)
			api.m.Invalid().Inc()
			continue
		}
		validAlerts = append(validAlerts, a)
    }
    // 保存告警
    api.alerts.Put(validAlerts...)
	// ...
	api.respond(w, nil)
}
```

开启http 服务:

```go
// 开启http 服务
srv := http.Server{Addr: *listenAddress, Handler: mux}
srv.ListenAndServe()
```


## 参考资料

- github.com/prometheus/alertmanager/api/api.go

