<!-- ---
title: 告警协调者实现
date: 2020-03-01 14:52:03
category: showcode, prometheus, alertmanager
--- -->

# 告警协调者实现

处理配置更新，有更新配置时会重新加载配置，并且重启内部子服务。

1. 配置协调器
2. 订阅处理
3. 重新载入配置协调器配置
4. 告警处理流程
5. 告警发送处理

![](images/alertmanager_coordinator.svg)

```go
// 配置协调器
configCoordinator := config.NewCoordinator(
    *configFile,
    prometheus.DefaultRegisterer,
    configLogger,
)

// 订阅处理
configCoordinator.Subscribe(func(conf *config.Config) error {
    // ...
})

// 重新载入配置协调器配置
configCoordinator.Reload()
```

## 1. 配置协调器

```go
// Coordinator 配置协调器
type Coordinator struct {
	configFilePath string
	logger         log.Logger

	// Protects config and subscribers
	mutex       sync.Mutex
	config      *Config
	subscribers []func(*Config) error
}

// 配置协调器
configCoordinator := config.NewCoordinator(
    *configFile,
    prometheus.DefaultRegisterer,
    configLogger,
)

// github.com/prometheus/alertmanager/config/coordinator.go
// NewCoordinator 配置协调器
func NewCoordinator(configFilePath string, r prometheus.Registerer, l log.Logger) *Coordinator {
	c := &Coordinator{
		configFilePath: configFilePath,
		logger:         l,
	}
    // ...
	return c
}
```

## 2. 订阅处理

```go
// 订阅处理
configCoordinator.Subscribe(func(conf *config.Config) error {})

// Subscribe subscribes the given Subscribers to configuration changes.
func (c *Coordinator) Subscribe(ss ...func(*Config) error) {
	// ...
	c.subscribers = append(c.subscribers, ss...)
}
```

## 3. 重新载入配置协调器配置

```go
// 重新载入配置协调器配置
configCoordinator.Reload()

// Reload 重新加载配置文件
func (c *Coordinator) Reload() error {
	// 重新加载配置
	if err := c.loadFromFile(); err != nil {
        // ...
		return err
	}
    
    // 通知订阅者配置更新
	if err := c.notifySubscribers(); err != nil {
		// ...
		return err
	}

	return nil
}

// loadFromFile 读取文件解析配置并且更新配置
func (c *Coordinator) loadFromFile() error {
	conf, err := LoadFile(c.configFilePath)
    // ...

	c.config = conf
	// ...

	return nil
}

// 使用新的配置，重新调用配置订阅方
func (c *Coordinator) notifySubscribers() error {
	for _, s := range c.subscribers {
		if err := s(c.config); err != nil {
			return err
		}
	}

	return nil
}
```

## 4. 告警处理流程

告警发送处理实现:

```go
// 根据配置文件创建告警接收者实例
for _, rcv := range conf.Receivers {
	// ...
	// 创建接收者实例
	integrations, err := buildReceiverIntegrations(rcv, tmpl, logger)
	receivers[rcv.Name] = integrations
}

// 创建流式处理器，逐个对告警进行处理
pipeline := pipelineBuilder.New(
	receivers,
	waitFunc,
	inhibitor,
	silencer,
	notificationLog,
	peer,
)

// 创建分发器
disp = dispatch.NewDispatcher(alerts, routes, pipeline, marker, timeoutFunc, logger, dispMetrics)

// 运行分发器，从内存告警存储实例中获取告警，并且在收到告警时，调用流式处理器对告警处理
// 处理调用 Exce 接口
go disp.Run()
```

创建接收器实例实现：

```go
// 接收器实例需要实现的接口
type Notifier interface {
	Notify(context.Context, ...*types.Alert) (bool, error)
}

// 创建接收器实例实现
func buildReceiverIntegrations(nc *config.Receiver, tmpl *template.Template, logger log.Logger) ([]notify.Integration, error) {
	var (
		errs         types.MultiError
		integrations []notify.Integration
		add          = func(name string, i int, rs notify.ResolvedSender, f func(l log.Logger) (notify.Notifier, error)) {
			// 创建接收器实例
			n, err := f(log.With(logger, "integration", name))
			integrations = append(integrations, notify.NewIntegration(n, rs, name, i))
		}
	)

	// webhook 接收器实例
	for i, c := range nc.WebhookConfigs {
		add("webhook", i, c, func(l log.Logger) (notify.Notifier, error) { return webhook.New(c, tmpl, l) })
	}
	// 电子邮件接收器
	for i, c := range nc.EmailConfigs {
		add("email", i, c, func(l log.Logger) (notify.Notifier, error) { return email.New(c, tmpl, l), nil })
	}
	// ...
	return integrations, nil
}
```

## 5. 告警发送处理

告警发送处理会经过多重策略进行处理。

```go
// 流式处理器实现
// 返回接收者策略
// github.com/prometheus/alertmanager/notify/notify.go
func (pb *PipelineBuilder) New(
	receivers map[string][]Integration,
	wait func() time.Duration,
	inhibitor *inhibit.Inhibitor,
	silencer *silence.Silencer,
	notificationLog NotificationLog,
	peer *cluster.Peer,
) RoutingStage {
	// 发送告警到接收器策略
	rs := make(RoutingStage, len(receivers))

	// 集群消息处理策略
	ms := NewGossipSettleStage(peer)
	// 抑制策略
	is := NewMuteStage(inhibitor)
	// 静默策略
	ss := NewMuteStage(silencer)

	// 针对每个接收者，创建接收者策略
	for name := range receivers {
		st := createReceiverStage(name, receivers[name], wait, notificationLog, pb.metrics)
		// 策略按照顺序执行，最后执行的是发送策略
		rs[name] = MultiStage{ms, is, ss, st}
	}
	return rs
}

// createReceiverStage 为接收者创建流式处理策略
func createReceiverStage(
	name string,
	integrations []Integration,
	wait func() time.Duration,
	notificationLog NotificationLog,
	metrics *metrics,
) Stage {
	var fs FanoutStage
	for i := range integrations {
		recv := &nflogpb.Receiver{
			GroupName:   name,
			Integration: integrations[i].Name(),
			Idx:         uint32(integrations[i].Index()),
		}
		// 增加发送重试，等待等策略
		var s MultiStage
		s = append(s, NewWaitStage(wait))
		s = append(s, NewDedupStage(&integrations[i], notificationLog, recv))
		s = append(s, NewRetryStage(integrations[i], name, metrics))
		s = append(s, NewSetNotifiesStage(notificationLog, recv))

		fs = append(fs, s)
	}
	return fs
}

// 接收者策略，获取接收者名称
func (rs RoutingStage) Exec(ctx context.Context, l log.Logger, alerts ...*types.Alert) (context.Context, []*types.Alert, error) {
	// 获取接收者名称
	receiver, ok := ReceiverName(ctx)
	
	// 取出接收者
	s, ok := rs[receiver]
	
	// 执行接收者策略
	return s.Exec(ctx, l, alerts...)
}

// Exec implements the Stage interface.
// 顺序执行：集群策略，抑制策略，静默策略和发送策略
func (ms MultiStage) Exec(ctx context.Context, l log.Logger, alerts ...*types.Alert) (context.Context, []*types.Alert, error) {
	var err error
	for _, s := range ms {
		if len(alerts) == 0 {
			return ctx, nil, nil
		}

		// 执行策略
		ctx, alerts, err = s.Exec(ctx, l, alerts...)
		if err != nil {
			return ctx, nil, err
		}
	}
	return ctx, alerts, nil
}

// 执行最后发送策略
func (fs FanoutStage) Exec(ctx context.Context, l log.Logger, alerts ...*types.Alert) (context.Context, []*types.Alert, error) {
	// 依次执行告警发送等待，发送重试等策略
	for _, s := range fs {
		go func(s Stage) {
			s.Exec(ctx, l, alerts...)
		}(s)
	}
	// ...
	return ctx, alerts, nil
}


// Exec 重发策略中，对策略进行发送
func (r RetryStage) Exec(ctx context.Context, l log.Logger, alerts ...*types.Alert) (context.Context, []*types.Alert, error) {
	// ...
	retry, err := r.integration.Notify(ctx, sent...)
}
```

## 参考资料

- github.com/prometheus/alertmanager/config/coordinator.go