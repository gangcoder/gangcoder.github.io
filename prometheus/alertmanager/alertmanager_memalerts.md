<!-- ---
title: 告警内存存储实现
date: 2020-03-01 14:50:55
category: showcode, prometheus, alertmanager
--- -->

# 告警内存存储实现

告警内存存储用于告警中心存储接收到的告警，告警是临时存储在内存中。

1. 创建告警内存存储实例
2. 告警订阅接口
3. 告警新增
4. 告警获取

![](images/alertmanager_memalerts.svg)


告警内存存储使用示例：

```go
// 创建告警内存存储实例
// github.com/prometheus/alertmanager/cmd/alertmanager/main.go
alerts, err := mem.NewAlerts(context.Background(), marker, *alertGCInterval, logger)

// 订阅告警
alerts.Subscribe()
```

## 1. 创建告警内存存储实例

```go
// Alerts 告警数据暂存
type Alerts interface {
	// Subscribe 获取所有告警的迭代器
	Subscribe() AlertIterator
	// GetPending 获取Pending 状态的告警迭代器
	GetPending() AlertIterator
	// Get 查询一个告警
	Get(model.Fingerprint) (*types.Alert, error)
	// Put 添加一个告警
	Put(...*types.Alert) error
}

// 告警访问提供者
type Alerts struct {
	alerts    *store.Alerts
	listeners map[int]listeningAlerts
	next      int
}

// store.Alerts
// 告警存储器
type Alerts struct {
	sync.Mutex
	c  map[model.Fingerprint]*types.Alert
	cb func([]*types.Alert)
}
```

创建告警内存存储实例：

```go
// 创建告警内存存储实例
// github.com/prometheus/alertmanager/cmd/alertmanager/main.go
alerts, err := mem.NewAlerts(context.Background(), marker, *alertGCInterval, logger)

// github.com/prometheus/alertmanager/provider/mem/mem.go
// NewAlerts 创建告警提供者
func NewAlerts(ctx context.Context, m types.Marker, intervalGC time.Duration, l log.Logger) (*Alerts, error) {
	// 创建内存告警存储器
	a := &Alerts{
		alerts:    store.NewAlerts(),
		cancel:    cancel,
		listeners: map[int]listeningAlerts{},
		next:      0,
		logger:    log.With(l, "component", "provider"),
	}
	a.alerts.SetGCCallback(func(alerts []*types.Alert) {
		for _, alert := range alerts {
			// 清理过期告警数据
			m.Delete(alert.Fingerprint())
		}

		a.mtx.Lock()
		// 清理退出的监听者
		for i, l := range a.listeners {
			select {
			case <-l.done:
				delete(a.listeners, i)
				close(l.alerts)
			default:
				// listener is not closed yet, hence proceed.
			}
		}
		a.mtx.Unlock()
	})
	go a.alerts.Run(ctx, intervalGC)

	return a, nil
}

// NewAlerts 告警存储器
func NewAlerts() *Alerts {
	a := &Alerts{
		c:  make(map[model.Fingerprint]*types.Alert),
		cb: func(_ []*types.Alert) {},
	}

	return a
}

// 告警存储器回收函数
func (a *Alerts) SetGCCallback(cb func([]*types.Alert)) {
	a.Lock()
	defer a.Unlock()

	a.cb = cb
}

// 定期运行存储器的回收函数
func (a *Alerts) Run(ctx context.Context, interval time.Duration) {
	t := time.NewTicker(interval)
	defer t.Stop()
	for {
		select {
		case <-ctx.Done():
			return
		case <-t.C:
			a.gc()
		}
	}
}

// 回收实现
func (a *Alerts) gc() {
	a.Lock()
	defer a.Unlock()

	// 收集已经解决的告警
	var resolved []*types.Alert
	for fp, alert := range a.c {
		if alert.Resolved() {
			delete(a.c, fp)
			resolved = append(resolved, alert)
		}
	}
	a.cb(resolved)
}
```

## 2. 告警订阅接口

```go
// 订阅告警
alerts.Subscribe()

// 返回一个新的订阅迭代器
func (a *Alerts) Subscribe() provider.AlertIterator {
	var (
		done   = make(chan struct{})
		alerts = a.alerts.List()
		ch     = make(chan *types.Alert, max(len(alerts), alertChannelLength))
	)

	// 告警写入channel
	for _, a := range alerts {
		ch <- a
	}

	// 记录订阅者，可以再有新告警时进行通知
	// 同时订阅者退出后会清理掉
	a.listeners[a.next] = listeningAlerts{alerts: ch, done: done}
	a.next++

	// 返回基于channel 的迭代器
	return provider.NewAlertIterator(ch, done, nil)
}

// List 返回所有暂存的告警数据
func (a *Alerts) List() []*types.Alert {
	// ...
	for _, alert := range a.c {
		alerts = append(alerts, alert)
	}

	return alerts
}
```


## 3. 告警新增

```go
func (a *Alerts) Put(alerts ...*types.Alert) error {
	// 告警存入内存存储器
	for _, alert := range alerts {
		// 告警告警唯一标签值
		fp := alert.Fingerprint()

		// 判断告警是否已经存在，如果存在就合并告警信息
		// 如果告警不存在会存储内存存储
		if err := a.alerts.Set(alert); err != nil {
			continue
		}

		a.mtx.Lock()
		// 通知订阅者
		for _, l := range a.listeners {
			select {
			case l.alerts <- alert:
			case <-l.done:
			}
		}
		a.mtx.Unlock()
	}

	return nil
}

// 内存存储写入告警信息
func (a *Alerts) Set(alert *types.Alert) error {
	// ...
	a.c[alert.Fingerprint()] = alert
	return nil
}
```


## 4. 告警获取


```go
// github.com/prometheus/alertmanager/provider/mem/mem.go
// Get 根据标签查询一个告警
func (a *Alerts) Get(fp model.Fingerprint) (*types.Alert, error) {
	return a.alerts.Get(fp)
}

func (a *Alerts) Get(fp model.Fingerprint) (*types.Alert, error) {
	// 加锁
	// ...
	alert, prs := a.c[fp]
	if !prs {
		return nil, ErrNotFound
	}
	return alert, nil
}
```

## 参考资料

- github.com/prometheus/alertmanager/provider/provider.go