<!-- ---
title: 告警抑制实现
date: 2020-03-01 14:52:39
category: showcode, prometheus, alertmanager
--- -->

# 告警抑制实现

抑制器用于一组告警出现时，可以抑制逻辑上相同的其他子告警，从而避免子告警通知信息泛滥。

1. 创建告警抑制器
2. 使用抑制器，在流式处理器中使用
3. 抑制判断
4. 运行抑制器

![](images/alertmanager_inhibitor.svg)

使用示例:

```go
// 创建告警抑制器
inhibitor = inhibit.NewInhibitor(alerts, conf.InhibitRules, marker, logger)

// 流式处理器中使用抑制器
// 抑制判断
inhibitor.Mutes(labels)

// 运行抑制器
go inhibitor.Run()
```

## 1. 创建告警抑制器

```go
// Inhibitor 基于抑制规则，判定给出的标签是否需要静默
type Inhibitor struct {
	alerts provider.Alerts
	rules  []*InhibitRule
	marker types.Marker
}

// 抑制规则
type InhibitRule struct {
	// 原告警匹配器
	SourceMatchers types.Matchers
	// 目标告警匹配器
	TargetMatchers types.Matchers
	Equal map[model.LabelName]struct{}
	scache *store.Alerts
}

// 创建告警抑制器
inhibitor = inhibit.NewInhibitor(alerts, conf.InhibitRules, marker, logger)

// github.com/prometheus/alertmanager/inhibit/inhibit.go
// NewInhibitor returns a new Inhibitor.
func NewInhibitor(ap provider.Alerts, rs []*config.InhibitRule, mk types.Marker, ...) *Inhibitor {
	ih := &Inhibitor{
		alerts: ap,
		marker: mk,
		logger: logger,
	}
	for _, cr := range rs {
		r := NewInhibitRule(cr)
		ih.rules = append(ih.rules, r)
	}
	return ih
}


// NewInhibitRule 读取配置的规则
func NewInhibitRule(cr *config.InhibitRule) *InhibitRule {
    // ...
    // 读取配置规则
	for ln, lv := range cr.SourceMatch {
		sourcem = append(sourcem, types.NewMatcher(model.LabelName(ln), lv))
	}
	for ln, lv := range cr.SourceMatchRE {
		sourcem = append(sourcem, types.NewRegexMatcher(model.LabelName(ln), lv.Regexp))
	}

	for ln, lv := range cr.TargetMatch {
		targetm = append(targetm, types.NewMatcher(model.LabelName(ln), lv))
	}
	for ln, lv := range cr.TargetMatchRE {
		targetm = append(targetm, types.NewRegexMatcher(model.LabelName(ln), lv.Regexp))
	}

	equal := map[model.LabelName]struct{}{}
	for _, ln := range cr.Equal {
		equal[ln] = struct{}{}
	}

	return &InhibitRule{
		SourceMatchers: sourcem,
		TargetMatchers: targetm,
		Equal:          equal,
		scache:         store.NewAlerts(),
	}
}
```

## 2. 使用抑制器

```go
// 流式处理器中使用抑制器
// 弱音器接口，用于判断告警是否需要禁声
type Muter interface {
	Mutes(model.LabelSet) bool
}

// github.com/prometheus/alertmanager/notify/notify.go
is := NewMuteStage(inhibitor)

// NewMuteStage return a new MuteStage.
func NewMuteStage(m types.Muter) *MuteStage {
	return &MuteStage{muter: m}
}

// Exec 判断告警经过抑制器后是否禁声
func (n *MuteStage) Exec(ctx context.Context, l log.Logger, alerts ...*types.Alert) (context.Context, []*types.Alert, error) {
	var filtered []*types.Alert
	for _, a := range alerts {
		// 判断是否禁声，没有禁声的继续返回
		if !n.muter.Mutes(a.Labels) {
			filtered = append(filtered, a)
		}
	}
	return ctx, filtered, nil
}
```


## 3. 抑制判断

```go
// 抑制判断
inhibitor.Mutes(labels)
// github.com/prometheus/alertmanager/inhibit/inhibit.go
// Mutes 判断是否需要抑制处理
func (ih *Inhibitor) Mutes(lset model.LabelSet) bool {
    // 标签对签名
	fp := lset.Fingerprint()

    // 逐一判断规则
	for _, r := range ih.rules {
        // 是否匹配目标告警规则
		if !r.TargetMatchers.Match(lset) {
			// If target side of rule doesn't match, we don't need to look any further.
			continue
		}
		// 是否匹配源告警规则
		if inhibitedByFP, eq := r.hasEqual(lset, r.SourceMatchers.Match(lset)); eq {
            // 设置抑制
			ih.marker.SetInhibited(fp, inhibitedByFP.String())
			return true
		}
    }
	return false
}
```

## 4. 运行抑制器

```go
// 运行抑制器
go inhibitor.Run()
// github.com/prometheus/alertmanager/inhibit/inhibit.go
// Run the Inhibitor's background processing.
func (ih *Inhibitor) Run() {
    // 运行抑制规则的内存告警存储规则
	for _, rule := range ih.rules {
		go rule.scache.Run(runCtx, 15*time.Minute)
	}

    // 运行抑制器
	g.Add(func() error {
		ih.run(runCtx)
		return nil
	}, func(err error) {
		runCancel()
	})
}

// 定时获取所有告警信息
// 判断告警是否满足抑制规则，如果满足抑制规则就暂存告警
func (ih *Inhibitor) run(ctx context.Context) {
	it := ih.alerts.Subscribe()

	for {
		select {
		case a := <-it.Next():
			// 判断告警是否满足抑制规则，如果满足抑制规则就暂存告警
			for _, r := range ih.rules {
				if r.SourceMatchers.Match(a.Labels) {
					r.scache.Set(a)
				}
			}
		}
	}
}
```



## 参考资料

- github.com/prometheus/alertmanager/inhibit/inhibit.go