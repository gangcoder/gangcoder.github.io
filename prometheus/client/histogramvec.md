<!-- ---
title: Histogram Vec
date: 2020-02-17 14:44:06
category: showcode, prometheus, client
--- -->

# Histogram Vec


## 5. Vec 指标

一系列相同Desc Histogram 指标处理

1. 创建实例
1. 添加指标数据
1. 数据收集

```go
// HistogramVec 包含大量相同Desc 的Histogram 指标，这些指标有着不同的标签值
// HistogramVec 基于metricVec 实现
type HistogramVec struct {
	*metricVec
}

// NewHistogramVec 基于提供的参数创建一个新的HistogramVec，指标之间后面会根据标签名对应的标签值区分
func NewHistogramVec(opts HistogramOpts, labelNames []string) *HistogramVec {
	desc := NewDesc(
		BuildFQName(opts.Namespace, opts.Subsystem, opts.Name),
		opts.Help,
		labelNames,
		opts.ConstLabels,
	)
	return &HistogramVec{
		metricVec: newMetricVec(desc, func(lvs ...string) Metric {
			return newHistogram(desc, opts, lvs...)
		}),
	}
}

// GetMetricWithLabelValues 使用标签值获取对应的Histogram 指标
// 如果是第一次查询，会根据标签值创建一个新的Histogram 指标
// 需要注意这里返回的是 Observer 类型数据，这种类型现在只有Histogram 和summary
func (v *HistogramVec) GetMetricWithLabelValues(lvs ...string) (Observer, error) {
	metric, err := v.metricVec.getMetricWithLabelValues(lvs...)
	if metric != nil {
		return metric.(Observer), err
	}
	return nil, err
}

// Collect vec 包含多个metric 指标实例，Collect 使用metricMap 提供的Collect
func (m *metricMap) Collect(ch chan<- Metric) {
	// 取出多个metric 实例
	for _, metrics := range m.metrics {
		for _, metric := range metrics {
			ch <- metric.metric
		}
	}
}
```

Observer 接口，Histogram 和summary 都实现了 Observer 接口

```go
// Observer 包含Observe 的接口
type Observer interface {
	Observe(float64)
}

// The ObserverFunc 适配器函数
type ObserverFunc func(float64)

// Observe calls f(value) 实现Observe 接口
func (f ObserverFunc) Observe(value float64) {
	f(value)
}
```

## 参考资料

- []()