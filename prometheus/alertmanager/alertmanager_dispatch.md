<!-- ---
title: 分发器
date: 2020-03-02 09:34:53
category: showcode, prometheus, alertmanager
--- -->

# 分发器实现

用于将告警分发给配置的接收者。

1. 创建告警分发器，将告警分发给所有接收器
2. 运行分发器
3. 告警分组路由

![](images/alertmanager_dispatch.svg)

```go
// 读取配置中的路由分组
routes := dispatch.NewRoute(conf.Route, nil)
activeReceivers := make(map[string]struct{})
routes.Walk(func(r *dispatch.Route) {
    activeReceivers[r.RouteOpts.Receiver] = struct{}{}
})

// 告警分发器，将告警分发给所有接收器
disp = dispatch.NewDispatcher(alerts, routes, pipeline, marker, timeoutFunc, logger, dispMetrics)

// 运行分发器
go disp.Run()
```

## 1. 创建告警分发器，将告警分发给所有接收器

```go
// Dispatcher 将进入的告警收集分组，并且分配给相应的通知对象
type Dispatcher struct {
	route   *Route
	alerts  provider.Alerts
	stage   notify.Stage
	metrics *DispatcherMetrics

	marker  types.Marker
	timeout func(time.Duration) time.Duration

	aggrGroups map[*Route]map[model.Fingerprint]*aggrGroup
}

// aggrGroup 告警发送分组
type aggrGroup struct {
	labels   model.LabelSet
	opts     *RouteOpts
	logger   log.Logger
	routeKey string

	alerts  *store.Alerts
}

// 告警分发器，将告警分发给所有接收器
disp = dispatch.NewDispatcher(alerts, routes, pipeline, marker, timeoutFunc, logger, dispMetrics)

// github.com/prometheus/alertmanager/dispatch/dispatch.go
// NewDispatcher 创建新的分组器
func NewDispatcher(
	ap provider.Alerts,
	r *Route,
	s notify.Stage,
	mk types.Marker,
	to func(time.Duration) time.Duration,
	l log.Logger,
	m *DispatcherMetrics,
) *Dispatcher {
	disp := &Dispatcher{
		alerts:  ap,
		stage:   s,
		route:   r,
		marker:  mk,
		timeout: to,
		logger:  log.With(l, "component", "dispatcher"),
		metrics: m,
	}
	return disp
}
```


## 2. 运行分发器


```go
// 运行分发器
go disp.Run()
// github.com/prometheus/alertmanager/dispatch/dispatch.go
// Run starts dispatching alerts incoming via the updates channel.
func (d *Dispatcher) Run() {
	// ...
	d.run(d.alerts.Subscribe())
}

func (d *Dispatcher) run(it provider.AlertIterator) {
    // ...
	for {
		select {
		case alert, ok := <-it.Next():
			// ...
			for _, r := range d.route.Match(alert.Labels) {
				d.processAlert(alert, r)
			}
		case <-cleanup.C:
			for _, groups := range d.aggrGroups {
				for _, ag := range groups {
					if ag.empty() {
						ag.stop()
						delete(groups, ag.fingerprint())
					}
				}
			}
		case <-d.ctx.Done():
			return
		}
	}
}

// processAlert 确定告警在那个告警分组中
func (d *Dispatcher) processAlert(alert *types.Alert, route *Route) {
    // 查找路由中的告警分组标签
	groupLabels := getGroupLabels(alert, route)
	fp := groupLabels.Fingerprint()

    // 获取路由已有告警分组
	group, ok := d.aggrGroups[route]
	if !ok {
		group = map[model.Fingerprint]*aggrGroup{}
		d.aggrGroups[route] = group
	}

	// 检查当前告警分组是否在已有分组中
	ag, ok := group[fp]
	if !ok {
        // 如果不在就创建新的告警分组
		ag = newAggrGroup(d.ctx, groupLabels, route, d.timeout, d.logger)
		group[fp] = ag

        // 运行告警分组
		go ag.run(func(ctx context.Context, alerts ...*types.Alert) bool {
            // 执行告警发送策略
			_, _, err := d.stage.Exec(ctx, d.logger, alerts...)
			// ...
			return err == nil
		})
	}
    // 最后将告警插入告警分组
	ag.insert(alert)
}
```

告警分组任务：创建告警组，运行告警组（会发送告警），告警插入告警组

```go
// newAggrGroup returns a new aggregation group.
func newAggrGroup(ctx context.Context, labels model.LabelSet, r *Route, to func(time.Duration) time.Duration, logger log.Logger) *aggrGroup {
	// ...
	ag := &aggrGroup{
		labels:   labels,
		routeKey: r.Key(),
		opts:     &r.RouteOpts,
		timeout:  to,
		alerts:   store.NewAlerts(),
		done:     make(chan struct{}),
	}

	return ag
}

func (ag *aggrGroup) run(nf notifyFunc) {
    // ...
	for {
		select {
		case now := <-ag.next.C:
			// 定时执行告警流式发送任务
			ag.flush(func(alerts ...*types.Alert) bool {
				return nf(ctx, alerts...)
			})
            // ...
		case <-ag.ctx.Done():
			return
		}
	}
}

// flush 通过notify 函数，发送所有告警信息
func (ag *aggrGroup) flush(notify func(...*types.Alert) bool) {
	// ...
	var (
		alerts      = ag.alerts.List() // 查询所有告警信息
		alertsSlice = make(types.AlertSlice, 0, len(alerts))
		now         = time.Now()
    )

    // 发送告警
	if notify(alertsSlice...) {
		for _, a := range alertsSlice {
            // 告警发送成功后，逐条清理告警
			if a.Resolved() && got.UpdatedAt == a.UpdatedAt {
                // ...
				ag.alerts.Delete(fp)
			}
		}
	}
}

// insert 告警插入告警组
func (ag *aggrGroup) insert(alert *types.Alert) {
	if err := ag.alerts.Set(alert); err != nil {
		level.Error(ag.logger).Log("msg", "error on set alert", "err", err)
	}
    // ...
}
```

## 3. 告警分组路由

```go
routes := dispatch.NewRoute(conf.Route, nil)

// github.com/prometheus/alertmanager/dispatch/route.go
// NewRoute 根据路由配置创建路由树
func NewRoute(cr *config.Route, parent *Route) *Route {
    // 读取匹配规则
    for ln, lv := range cr.Match {
		matchers = append(matchers, types.NewMatcher(model.LabelName(ln), lv))
	}
	for ln, lv := range cr.MatchRE {
		matchers = append(matchers, types.NewRegexMatcher(model.LabelName(ln), lv.Regexp))
	}
	sort.Sort(matchers)

    // 创建路由实例
	route := &Route{
		parent:    parent,
		RouteOpts: opts,
		Matchers:  matchers,
		Continue:  cr.Continue,
	}

	route.Routes = NewRoutes(cr.Routes, route)
	return route
}


// Match 在路由树中匹配分组路由
func (r *Route) Match(lset model.LabelSet) []*Route {
    // ...
	var all []*Route
    // 匹配规则
	for _, cr := range r.Routes {
		matches := cr.Match(lset)

		all = append(all, matches...)

		if matches != nil && !cr.Continue {
			break
		}
	}
    // ...
	return all
}
```

## 参考资料

- github.com/prometheus/alertmanager/dispatch/dispatch.go