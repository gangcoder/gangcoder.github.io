<!-- ---
title: SummaryVec
date: 2020-02-14 08:01:50
category: showcode, prometheus, client
--- -->

# SummaryVec


## 5. Vec 指标

基于metricVec 实现 SummaryVec。

```go
// SummaryVec 收集一系列 Summaries
type SummaryVec struct {
	*metricVec
}

// NewSummaryVec
func NewSummaryVec(opts SummaryOpts, labelNames []string) *SummaryVec {
	for _, ln := range labelNames {
		if ln == quantileLabel {
			panic(errQuantileLabelNotAllowed)
		}
	}
	desc := NewDesc(
		BuildFQName(opts.Namespace, opts.Subsystem, opts.Name),
		opts.Help,
		labelNames,
		opts.ConstLabels,
	)
	return &SummaryVec{
		metricVec: newMetricVec(desc, func(lvs ...string) Metric {
			return newSummary(desc, opts, lvs...)
		}),
	}
}

// newMetricVec returns an initialized metricVec.
func newMetricVec(desc *Desc, newMetric func(lvs ...string) Metric) *metricVec {
	return &metricVec{
		metricMap: &metricMap{
			metrics:   map[uint64][]metricWithLabelValues{},
			desc:      desc,
			newMetric: newMetric,
		},
		hashAdd:     hashAdd,
		hashAddByte: hashAddByte,
	}
}

// WithLabelValues 通过标签值获取Metric 指标实例
//myVec.WithLabelValues("404", "GET").Observe(42.21)
func (v *SummaryVec) WithLabelValues(lvs ...string) Observer {
	s, err := v.GetMetricWithLabelValues(lvs...)
	if err != nil {
		panic(err)
	}
	return s
}

func (v *SummaryVec) GetMetricWithLabelValues(lvs ...string) (Observer, error) {
	metric, err := v.metricVec.getMetricWithLabelValues(lvs...)
	if metric != nil {
		return metric.(Observer), err
	}
	return nil, err
}

// Collect 实现数据收集接口，获取所有Metric 指标数据
func (m *metricMap) Collect(ch chan<- Metric) {
	m.mtx.RLock()
	defer m.mtx.RUnlock()

	for _, metrics := range m.metrics {
		for _, metric := range metrics {
			ch <- metric.metric
		}
	}
}
```


## 参考资料

- []()