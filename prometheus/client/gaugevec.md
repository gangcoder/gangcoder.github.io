<!-- ---
title: GaugeVec
date: 2020-02-14 01:51:52
category: showcode, prometheus, client
--- -->

# GaugeVec



## GaugeVec 指标

GaugeVec 和CounterVec 的处理方式一样，都是基于 `metricVec` 进行处理。

```go
// GaugeVec 是一系列Gauge 指标，他们通过不同的标签值进行区分
type GaugeVec struct {
	*metricVec
}

// NewGaugeVec 创建一个 GaugeVec，这些Gauge 通过label 进行区分
func NewGaugeVec(opts GaugeOpts, labelNames []string) *GaugeVec {
	desc := NewDesc(
		BuildFQName(opts.Namespace, opts.Subsystem, opts.Name),
		opts.Help,
		labelNames,
		opts.ConstLabels,
	)
	return &GaugeVec{
		metricVec: newMetricVec(desc, func(lvs ...string) Metric {
			if len(lvs) != len(desc.variableLabels) {
				panic(makeInconsistentCardinalityError(desc.fqName, desc.variableLabels, lvs))
			}
			result := &gauge{desc: desc, labelPairs: makeLabelPairs(desc, lvs)}
			result.init(result) // Init self-collection.
			return result
		}),
	}
}

// GetMetricWith 通过标签键值对查找对应的Gauge 指标，如果是第一次处理就会创建Gauge 指标
// 和所有GaugeVec 指标一样，这些操作是基于metricVec 内部函数进行处理
func (v *GaugeVec) GetMetricWith(labels Labels) (Gauge, error) {
	metric, err := v.metricVec.getMetricWith(labels)
	if metric != nil {
		return metric.(Gauge), err
	}
	return nil, err
}

// With 通过标签找到对应的Gauge 指标
// myVec.With(prometheus.Labels{"code": "404", "method": "GET"}).Add(42)
func (v *GaugeVec) With(labels Labels) Gauge {
	g, err := v.GetMetricWith(labels)
	if err != nil {
		panic(err)
	}
	return g
}
```

## 参考资料

- github.com/prometheus/client_golang/prometheus/gauge.go
