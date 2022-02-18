<!-- ---
title: struct
date: 2019-05-25 04:43:17
category: src, prometheus
--- -->

struct

```go
type Registry struct {
	mtx                   sync.RWMutex // 读写锁
	collectorsByID        map[uint64]Collector // ID is a hash of the descIDs. descId 的和
	descIDs               map[uint64]struct{} // descId
	dimHashesByName       map[string]uint64 // 全局唯一ID
	uncheckedCollectors   []Collector
	pedanticChecksEnabled bool
}

type MetricType int32

const (
	MetricType_COUNTER   MetricType = 0
	MetricType_GAUGE     MetricType = 1
	MetricType_SUMMARY   MetricType = 2
	MetricType_UNTYPED   MetricType = 3
	MetricType_HISTOGRAM MetricType = 4
)

type Metric struct {
	Label                []*LabelPair `protobuf:"bytes,1,rep,name=label" json:"label,omitempty"`
	Gauge                *Gauge       `protobuf:"bytes,2,opt,name=gauge" json:"gauge,omitempty"`
	Counter              *Counter     `protobuf:"bytes,3,opt,name=counter" json:"counter,omitempty"`
	Summary              *Summary     `protobuf:"bytes,4,opt,name=summary" json:"summary,omitempty"`
	Untyped              *Untyped     `protobuf:"bytes,5,opt,name=untyped" json:"untyped,omitempty"`
	Histogram            *Histogram   `protobuf:"bytes,7,opt,name=histogram" json:"histogram,omitempty"`
	TimestampMs          *int64       `protobuf:"varint,6,opt,name=timestamp_ms,json=timestampMs"`
}

type MetricFamily struct {
	Name                 *string     `protobuf:"bytes,1,opt,name=name" json:"name,omitempty"`
	Help                 *string     `protobuf:"bytes,2,opt,name=help" json:"help,omitempty"`
	Type                 *MetricType `protobuf:"varint,3,opt,name=type,enum=io.prometheus.client.MetricType"`
	Metric               []*Metric   `protobuf:"bytes,4,rep,name=metric" json:"metric,omitempty"`
}

type Collector interface {
    // Describe 将所有collector 收集的 metrics 写入到参数提供的 channel 中
	Describe(chan<- *Desc)
	// Collect 在prometheus 注册器收集metrics 信息时调用该函数；这里会将收集到的metrics 写入参数channel，并且一次返回
    // metrics 的descriptor 通过调用Describe 获得
	Collect(chan<- Metric)
}
```

## metric

```go
type counter struct {
	//数值存储
	valBits uint64
	valInt  uint64

	selfCollector
	desc *Desc

	labelPairs []*dto.LabelPair
}
```

## 参考资料

- []()