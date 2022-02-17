<!-- ---
title: prometheus notifier
date: 2018-11-26 11:29:19
category: src, prometheus, src
--- -->

# Notifier 告警通知实现

接收从 `rules` 规则解析器中采集到的Alert 告警信息，并且将告警信息推送到告警中心。

代码实现上包含以下4 个主要部分：

1. 创建 notifier 实例 `NewManager`
2. 更新 notifier 配置信息 `ApplyConfig` 
3. 运行 notifier 通知逻辑，主逻辑中实现：接收服务发现获取的告警中心地址；将alerts 推送到告警中心
4. 发送告警到 notifier，通过`NotifyFunc` 函数发送alerts 到notifier `n.queue`


![](images/prometheus_notifier.svg)


以上调用代码主要在 `cmd/prometheus/main.go` 中实现：

```go
// cmd/prometheus/main.go
notifierManager = notifier.NewManager(&cfg.notifier, log.With(logger, "component", "notifier"))
notifierManager.ApplyConfig()
NotifyFunc: sendAlerts(notifierManager, cfg.web.ExternalURL.String())
notifierManager.Run(discoveryManagerNotify.SyncCh())
```

## 1. 创建 notifier 实例

调用 `github.com/prometheus/prometheus/notifier/notifier.go` 中的 `NewManager` 函数，创建一个 notifier `Manager` 实例。

这里`Manager` 的数据结构相对复杂，其中最重要的是2 个字段 `queue`, `alertmanagers`：

- `queue` 是个 `Alert slice` ，他的作用类似于队列，用来暂存由 `rules` 解析出来的Alert；然后再由另一个goroutine 取出发送给告警中心
- `alertmanagers` 用来保存告警中心地址，告警中心地址通过服务发现进行更新

notifier Manager 数据结构：

```go
//notifier 发送管理实例
type Manager struct {
    queue []*Alert // 暂存Alert
    more   chan struct{} // `rules` 解析出新的Alert 时，向more 写入一个chan，触发Alert 发送到告警中心
    alertmanagers map[string]*alertmanagerSet // 告警中心地址合集
}

// alertmanagerSet 告警中心群组，包含一组服务器地址
type alertmanagerSet struct {
    client *http.Client // http client 用于发送Alert
    ams        []alertmanager // 具体的某一个告警中心地址
    droppedAms []alertmanager // 过期下线的地址
}

// alertmanager 告警中心地址
type alertmanager interface {
    url() *url.URL // 获取告警中心地址URL
}
```

创建 notifier 实例：

```go
//prometheus/cmd/prometheus/main.go
notifierManager = notifier.NewManager(&cfg.notifier, log.With(logger, "component", "notifier"))

// 创建notifier 实例
func NewManager(o *Options, logger log.Logger) *Manager {
    // ...
    // 创建Manager
    n := &Manager{
        queue:  make([]*Alert, 0, o.QueueCapacity),
        more:   make(chan struct{}, 1),
    }
    // ...
    return n
}
```

## 2. 更新 notifier 配置

`prometheus server main.go` 代码中，通过 `reloaders` 函数载入配置信息。在 notifier 中，会根据配置创建告警中心群组实例 `n.alertmanagers`

```go
//notifier.ApplyConfig

// ApplyConfig 使用新配置
func (n *Manager) ApplyConfig(conf *config.Config) error {
    // ...
    amSets := make(map[string]*alertmanagerSet)

    // 根据告警中心配置，创建告警中心
    for k, cfg := range conf.AlertingConfig.AlertmanagerConfigs.ToMap() {
        ams, err := newAlertmanagerSet(cfg, n.logger, n.metrics)
        
        amSets[k] = ams
    }

    // alerts
    n.alertmanagers = amSets

    return nil
}

// newAlertmanagerSet 告警中心群组
func newAlertmanagerSet(cfg *config.AlertmanagerConfig, logger log.Logger, metrics *alertMetrics) (*alertmanagerSet, error) {
	// 将Alert 发送到告警中心是通过http 请求，在这里初始化告警中心http client
    client, err := config_util.NewClientFromConfig(cfg.HTTPClientConfig, "alertmanager", false)
	
	s := &alertmanagerSet{
		client: client,
		cfg:    cfg,
		logger: logger,
	}
	return s, nil
}
```

## 3. 运行 notifier 通知逻辑

notifier Run 函数中主要做两件事情：获取最新告警中心地址，将告警信息发送到告警中心。

1. 从服务发现中获取最新告警中心地址
2. 通过`nextBatch` 函数获取一批告警通知信息，这里最终是从 `n.queue` 获取的Alerts
3. 向所有告警中心群组发送这些告警信息

```go
// prometheus/cmd/prometheus/main.go
notifier.Run(discoveryManagerNotify.SyncCh())

// Run 持续更新告警中心地址和分发告警通知
func (n *Manager) Run(tsets <-chan map[string][]*targetgroup.Group) {
    for {
        select { // 阻塞，当有下面3 者之一的chan 信号时才会继续处理
        case <-n.ctx.Done(): // 超时或者停止处理
            return
        case ts := <-tsets: // 告警中心地址更新消息，需要重新载入告警中心地址
            n.reload(ts)
        case <-n.more: // 有Alerts 接收到消息，需要处理下面的逻辑
        }

        // 当有Alerts 接收到时，获取要通知的警报消息
        alerts := n.nextBatch()

        // 将警报消息发送给告警中心
        if !n.sendAll(alerts...) {
            n.metrics.dropped.Add(float64(len(alerts)))
        }
        
        // 如果告警消息没有一次发送完成，通过setMore 向more 中写入一个chan 信息以开启下一次处理
        if n.queueLen() > 0 {
            n.setMore()
        }
    }
}
```

### 3.1 更新告警中心地址

从服务发现中获取最新的告警中心地址，并且合并到已有地址中，处理分为2 步实现：

1. 获取服务发现的 tgs 目标机器
2. 调用 `sync` 将新告警中心地址格式转为需要的格式，去重过滤后放入 `n.alertmanagers.ams` 中

```go
// 将获取到的消息中心地址并入已有地址中
func (n *Manager) reload(tgs map[string][]*targetgroup.Group) {
    for id, tgroup := range tgs {
        am, ok := n.alertmanagers[id]
        // 不在配置内的目标节点，不处理
        if !ok {
            // ...
            continue
        }
        am.sync(tgroup) // 如果是已经初始化的告警中心群组，就需要将地址并入
    }
}

// 将新告警中心地址格式转为需要的格式.
func (s *alertmanagerSet) sync(tgs []*targetgroup.Group) {
    // 运行中的地址
    allAms := []alertmanager{}
    // 失效的告警中心地址
    allDroppedAms := []alertmanager{}

    for _, tg := range tgs {
        // alertmanagerFromGroup 根据tg 的标签，将tg 目标分为两类
        // 正常状态的tg 和已经失效下线的tg
        ams, droppedAms, err := alertmanagerFromGroup(tg, s.cfg)
        allAms = append(allAms, ams...)
        allDroppedAms = append(allDroppedAms, droppedAms...)
    }

    // 对于一个告警中心群组，使用将新地址
    s.ams = []alertmanager{}
    // 已经失效的告警中心
    s.droppedAms = []alertmanager{}
    s.droppedAms = append(s.droppedAms, allDroppedAms...)

    // 正常告警中心地址过滤去重
    seen := map[string]struct{}{}
    for _, am := range allAms {
        us := am.url().String()
        if _, ok := seen[us]; ok {
            continue
        }

        seen[us] = struct{}{}
        s.ams = append(s.ams, am)
    }
}
```

### 3.2 发送告警消息

```go
// 获取要通知的警报消息
func (n *Manager) nextBatch() []*Alert {
    // ...
    var alerts []*Alert

	alerts = append(make([]*Alert, 0, len(n.queue)), n.queue...)
    n.queue = n.queue[:0]

	return alerts
}

// 将所有告警消息发送给消息中心
// 只要成功发送到一个告警中心服务就算发送成功
func (n *Manager) sendAll(alerts ...*Alert) bool {
    amSets := n.alertmanagers
    // ...
    // 获取所有告警中心服务器组
    for _, ams := range amSets {
        switch ams.cfg.APIVersion {
		case config.AlertmanagerAPIVersionV1:
            v1Payload, err = json.Marshal(alerts)
            payload = v1Payload     
		case config.AlertmanagerAPIVersionV2:
            openAPIAlerts := alertsToOpenAPIAlerts(alerts)
            v2Payload, err = json.Marshal(openAPIAlerts)
            payload = v2Payload
		default:
			return false
		}

        // 获取每组告警中心的每台服务器地址
        for _, am := range ams.ams {
            wg.Add(1)

            // 往每个服务器发送告警信息
            go func(ams *alertmanagerSet, am alertmanager) {
                // 告警中心服务器地址
                n.sendOne(ctx, client, url, payload)

                wg.Done()
            }(ams.client, am.url().String())
        }
    }
    wg.Wait()

    return numSuccess > 0
}

// 往一个告警中心服务器 endpoint 发送告警信息
func (n *Manager) sendOne(ctx context.Context, c *http.Client, url string, b []byte) error {
    req, err := http.NewRequest("POST", url, bytes.NewReader(b))

    req.Header.Set("Content-Type", contentTypeJSON)
	resp, err := n.opts.Do(ctx, c, req)
    defer resp.Body.Close()

    // Any HTTP status 2xx is OK.
    if resp.StatusCode/100 != 2 {
		return errors.Errorf("bad response status %s", resp.Status)
	}
    return err
}
```


## 4. 发送告警到 notifier

ruleManager 在解析到告警消息时，通过 `NotifyFunc` ，可以将解析到的Alert 写入 notifier 的队列 `n.queue` 中。

```go
// server main.go 中实现了 NotifyFunc
// NotifyFunc: sendAlerts(notifierManager, cfg.web.ExternalURL.String())

// rules 在解析到告警时发送告警到 notifier
func sendAlerts(s sender, externalURL string) rules.NotifyFunc {
	return func(ctx context.Context, expr string, alerts ...*rules.Alert) {
		var res []*notifier.Alert

        // 转换Alert 的格式
		for _, alert := range alerts {
			a := &notifier.Alert{
				StartsAt:     alert.FiredAt,
				Labels:       alert.Labels,
				Annotations:  alert.Annotations,
				GeneratorURL: externalURL + strutil.TableLinkForExpression(expr),
			}
			res = append(res, a)
		}

		if len(alerts) > 0 {
			n.Send(res...)
		}
	}
}

// Send 将alert 写入 queue 队列中
func (n *Manager) Send(alerts ...*Alert) {
	// ...
	// 调整Alert 的标签属性
	alerts = n.relabelAlerts(alerts)

    // ...
	// 如果queue 中已有的alerts 和新产生的alerts 超过最大量，则去掉前d 条数据
	if d := (len(n.queue) + len(alerts)) - n.opts.QueueCapacity; d > 0 {
		n.queue = n.queue[d:]
    }

    // 将Alert 放入暂存队列
	n.queue = append(n.queue, alerts...)

	// 通知将告警发送到告警中心的 goroutine ，有更多的alerts 进来
	n.setMore()
}
```


## 参考资料

- github.com/prometheus/prometheus/cmd/prometheus/main.go
- github.com/prometheus/prometheus/notifier/notifier.go

