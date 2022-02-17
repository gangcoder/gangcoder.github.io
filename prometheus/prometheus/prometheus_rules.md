<!-- ---
title: rules
date: 2019-04-18 11:05:41
category: src, prometheus, src
--- -->

# Rules 规则服务实现

规则服务根据域名配置的规则查询时序数据。规则有数据记录规则和告警规则。

1. 创建 rules 服务实例
2. 更新规则配置，根据规则配置创建规则实例，并且运行规则
3. 开启rules 服务

![](images/prometheus_rules.svg)

```go
//cmd/prometheus/main.go
// 创建实例
ruleManager = rules.NewManager(...)
// 更新配置
ruleManager.Update(...)
// 启动rule 
ruleManager.Run()
```

## 1. 创建 rules 服务实例

```go
// Rules 管理器，管理recording 和alerting 规则
type Manager struct {
	opts     *ManagerOptions
	groups   map[string]*Group
}

// rules 管理器配置
type ManagerOptions struct {
	ExternalURL     *url.URL
	QueryFunc       QueryFunc
	NotifyFunc      NotifyFunc
	Context         context.Context
	Appendable      Appendable
}

// Group 一组规则
type Group struct {
	name                 string
	file                 string
	rules                []Rule
    opts                 *ManagerOptions
}

// 一条规则需要实现的接口
type Rule interface {
	Name() string
	Labels() labels.Labels
	Eval(context.Context, time.Time, QueryFunc, *url.URL) (promql.Vector, error)
}

// 告警规则
type AlertingRule struct {
	// 告警名称
	name string
	// 告警表达式
    vector promql.Expr
    // 告警标签
	labels labels.Labels
	// 告警注释
	annotations labels.Labels
    // 当前激活中的告警
	active map[uint64]*Alert
}

// 普通告警规则
type RecordingRule struct {
	name   string
	vector promql.Expr
	labels labels.Labels
}
```

创建 rule 实例：

```go
// github.com/prometheus/prometheus/cmd/prometheus/main.go
ruleManager = rules.NewManager(&rules.ManagerOptions{
    Appendable:      fanoutStorage,
    TSDB:            localStorage,
    QueryFunc:       rules.EngineQueryFunc(queryEngine, fanoutStorage),
    NotifyFunc:      sendAlerts(notifierManager, cfg.web.ExternalURL.String()),
})

// NewManager 创建Rules 管理器，管理recording 和alerting 规则
func NewManager(o *ManagerOptions) *Manager {
    // ...
	m := &Manager{
		groups: map[string]*Group{},
		opts:   o,
		block:  make(chan struct{}),
	}

	return m
}
```

## 2. 更新规则配置

从配置文件中读取配置的规则。

```go
//github.com/prometheus/prometheus/cmd/prometheus/main.go
// 获取rules 的配置文件列表
for _, pat := range cfg.RuleFiles {
    fs, err := filepath.Glob(pat)
    // ...
    files = append(files, fs...)
}
// 更新rules 配置
return ruleManager.Update(
    time.Duration(cfg.GlobalConfig.EvaluationInterval),
    files,
    cfg.GlobalConfig.ExternalLabels,
)

// 解析groups，从rules 文件中读取rule
func (m *Manager) Update(interval time.Duration, files []string, externalLabels labels.Labels) error {
    // 读取配置中的规则
    groups, errs := m.LoadGroups(interval, externalLabels, files...)
    // ...

    // 执行规则
    for _, newg := range groups {
		go func(newg *Group) {
			go func() {
                // 规则开启运行
				<-m.block
				newg.run(m.opts.Context)
			}()
			wg.Done()
		}(newg)
	}

	// 旧规则停止
	for n, oldg := range m.groups {
		oldg.stop()
	}

    wg.Wait()
    // 更新规则
	m.groups = groups

	return nil
}
```

### 2.1 创建规则实例

读取配置中的规则后，基于规则配置创建规则实例。

```go
// 读取配置文件中的rule 规则
func (m *Manager) LoadGroups(
	interval time.Duration, externalLabels labels.Labels, filenames ...string,
) (map[string]*Group, []error) {
	groups := make(map[string]*Group)
    // ...
	for _, fn := range filenames {
        // 读取配置文件
		rgs, errs := rulefmt.ParseFile(fn)

        // 创建规则实例
		for _, rg := range rgs.Groups {
			rules := make([]Rule, 0, len(rg.Rules))
			for _, r := range rg.Rules {
                // 解析rules 规则的查询语句
				expr, err := promql.ParseExpr(r.Expr.Value)
                
                // 判断是否是告警规则
				if r.Alert.Value != "" {
					rules = append(rules, NewAlertingRule(
						r.Alert.Value,
						expr,
						// ...
					))
					continue
                }

                // 记录规则
				rules = append(rules, NewRecordingRule(
					r.Record.Value,
					expr,
					labels.FromMap(r.Labels),
				))
			}

            // 将规则分组，创建规则组结构
			groups[groupKey(fn, rg.Name)] = NewGroup(rg.Name, fn, itv, rules, shouldRestore, m.opts)
		}
	}

	return groups, nil
}

// NewAlertingRule 创建告警查询规则
func NewAlertingRule(
	name string, vec promql.Expr, hold time.Duration,
	labels, annotations, externalLabels labels.Labels,
	restored bool, logger log.Logger,
) *AlertingRule {
	el := make(map[string]string, len(externalLabels))
	for _, lbl := range externalLabels {
		el[lbl.Name] = lbl.Value
	}

	return &AlertingRule{
		name:           name,
		vector:         vec,
		labels:         labels,
		annotations:    annotations,
		active:         map[uint64]*Alert{},
        restored:       restored,
        // ...
	}
}

// NewRecordingRule 创建一条常规规则
func NewRecordingRule(name string, vector promql.Expr, lset labels.Labels) *RecordingRule {
	return &RecordingRule{
		name:   name,
		vector: vector,
		health: HealthUnknown,
		labels: lset,
	}
}

// NewGroup 一组规则
func NewGroup(name, file string, interval time.Duration, rules []Rule, shouldRestore bool, opts *ManagerOptions) *Group {
	// opts 使用的是 ruleManager 的opts，进而可以使用查询函数和通知函数
	return &Group{
		name:                 name,
		file:                 file,
		interval:             interval,
		rules:                rules,
		shouldRestore:        shouldRestore,
		opts:                 opts,
		// ...
	}
}
```

### 2.2 运行规则

规则会分组，所以先运行规则组。

```go
func (g *Group) run(ctx context.Context) {
    //...
    iter := func() {
        // ...
		g.Eval(ctx, evalTimestamp)
    }

    for {
		select {
		case <-g.done:
			return
		default:
			select {
			case <-g.done:
				return
			case <-tick.C:
				// ...
				iter()
			}
		}
	}
}

// Eval 执行规则组的求值
func (g *Group) Eval(ctx context.Context, ts time.Time) {
    //...
    for i, rule := range g.rules {
        func(i int, rule Rule) {
            vector, err := rule.Eval(ctx, ts, g.opts.QueryFunc, g.opts.ExternalURL)

            // 如果是告警规则，还需要发送告警信息
            if ar, ok := rule.(*AlertingRule); ok {
				ar.sendAlerts(ctx, ts, g.opts.ResendDelay, g.interval, g.opts.NotifyFunc)
            }

            app, err := g.opts.Appendable.Appender()
            for _, s := range vector {
                // 存储数据
                app.Add(s.Metric, s.T, s.V)
            }
            // 存储提交
            app.Commit()
        }(i, rule)
    }
}

// 告警规则执行
func (r *AlertingRule) Eval(ctx context.Context, ts time.Time, query QueryFunc, externalURL *url.URL) (promql.Vector, error) {
    // 调用查询函数，查询数据
    res, err := query(ctx, r.vector.String(), ts)

    // ...
    // 创建告警实例
    for _, smpl := range res {
        h := lbs.Hash()
        alerts[h] = &Alert{
			Labels:      lbs,
			Annotations: annotations,
			ActiveAt:    ts,
			State:       StatePending,
			Value:       smpl.V,
		}
    }
    
    // 判断是否是活跃告警
    for h, a := range alerts {
		// 如果活跃告警中有相同告警，则更新告警信息
		if alert, ok := r.active[h]; ok && alert.State != StateInactive {
			alert.Value = a.Value
			alert.Annotations = a.Annotations
			continue
		}

		r.active[h] = a
	}

	// 检查告警有效性
	for fp, a := range r.active {
		// ...
		if a.State == StatePending && ts.Sub(a.ActiveAt) >= r.holdDuration {
			a.State = StateFiring
			a.FiredAt = ts
		}

        // 调整数据结构
		if r.restored {
			vec = append(vec, r.sample(a, ts))
			vec = append(vec, r.forStateSample(a, ts, float64(a.ActiveAt.Unix())))
		}
	}

	// 返回告警数据
	return vec, nil
}

// 记录规则执行
func (rule *RecordingRule) Eval(ctx context.Context, ts time.Time, query QueryFunc, _ *url.URL) (promql.Vector, error) {
    // 通过查询函数，查询规则
	vector, err := query(ctx, rule.vector.String(), ts)
    
    // 格式化标签
	for i := range vector {
		sample := &vector[i]

		lb := labels.NewBuilder(sample.Metric)
		lb.Set(labels.MetricName, rule.name)
		for _, l := range rule.labels {
			lb.Set(l.Name, l.Value)
		}

		sample.Metric = lb.Labels()
	}

	// ...
	return vector, nil
}

// 发送告警规则到通知服务，再由通知服务将告警推送给告警中心
func (r *AlertingRule) sendAlerts(ctx context.Context, ts time.Time, resendDelay time.Duration, interval time.Duration, notifyFunc NotifyFunc) {
	// 调用通知函数
	notifyFunc(ctx, r.vector.String(), alerts...)
}
```

## 3. 开启rules 服务

开启rules 服务的方法是关闭阻塞 `channel`。

```go
// github.com/prometheus/prometheus/rules/manager.go
// Run 开始rule 处理
func (m *Manager) Run() {
	close(m.block)
}
```

## 4. 查询与通知函数

查询函数

```go
// 查询函数
// QueryFunc: rules.EngineQueryFunc(queryEngine, fanoutStorage),
// QueryFunc 处理PromQL 查询
type QueryFunc func(ctx context.Context, q string, t time.Time) (promql.Vector, error)
```

通知函数

```go
// 通知函数
// NotifyFunc: sendAlerts(notifierManager, cfg.web.ExternalURL.String()),
// 通知函数，将告警发送到通知服务
type NotifyFunc func(ctx context.Context, expr string, alerts ...*Alert)
```

## 参考资料

- github.com/prometheus/prometheus/cmd/prometheus/main.go
- github.com/prometheus/prometheus/rules/manager.go

