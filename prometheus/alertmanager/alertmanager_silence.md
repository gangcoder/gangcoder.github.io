<!-- ---
title: 告警静默器
date: 2020-03-01 14:49:33
category: showcode, prometheus, alertmanager
--- -->

# 告警静默器实现

用于对告警进行静默处理，这样当有人处理告警时，不至于让告警泛滥。

1. 创建静默器，处理静默规则
2. 创建静默器广播通道
3. 静默器维护处理，包括处理过期的静默，创建快照保存
4. 创建静默处理器，用于判断告警是否需要静默

![](images/alertmanager_silence.svg)

使用示例：

```go
// 创建silences 静默器
silences, err := silence.New(silenceOpts)

// 静默期广播通道
c := peer.AddState("sil", silences, prometheus.DefaultRegisterer)
silences.SetBroadcast(c.Broadcast)

// 静默器维护处理，包括处理过期的静默，创建快速保存
silences.Maintenance(15*time.Minute, filepath.Join(*dataDir, "silences"), stopc)

// 创建静默处理器
silencer := silence.NewSilencer(silences, marker, logger)

// 静默处理器更新静默状态
silencer.Mutes(labels)
```

## 1. 创建静默器

```go
// Silences 持有一个静默器
type Silences struct {
	retention time.Duration
	st        state
	broadcast func([]byte)
	mc        matcherCache
}

type state map[string]*pb.MeshSilence

// Silencer 静默器
type Silencer struct {
	silences *Silences
	marker   types.Marker
	logger   log.Logger
}

// 创建silences 静默器
silences, err := silence.New(silenceOpts)
// github.com/prometheus/alertmanager/silence/silence.go

// New returns a new Silences object with the given configuration.
func New(o Options) (*Silences, error) {
	s := &Silences{
		mc:        matcherCache{},
		logger:    log.NewNopLogger(),
		retention: o.Retention,
		now:       utcNow,
		broadcast: func([]byte) {},
		st:        state{},
	}
    
    // 加载快照数据
	if o.SnapshotReader != nil {
		if err := s.loadSnapshot(o.SnapshotReader); err != nil {
			return s, err
		}
	}
	return s, nil
}

// loadSnapshot 加载快照生成的state
func (s *Silences) loadSnapshot(r io.Reader) error {
	st, err := decodeState(r)
    // 写入state 中
	for _, e := range st {
		st[e.Silence.Id] = e
	}
	s.mtx.Lock()
	s.st = st
	s.version++
	s.mtx.Unlock()

	return nil
}
```

## 2. 创建静默器广播通道

```go
// 静默期广播通道
c := peer.AddState("sil", silences, prometheus.DefaultRegisterer)
silences.SetBroadcast(c.Broadcast)

// github.com/prometheus/alertmanager/silence/silence.go
// SetBroadcast 设置广播函数
func (s *Silences) SetBroadcast(f func([]byte)) {
    // ...
	s.broadcast = f
}
```

## 3. 静默器维护处理


```go
// 静默器维护处理，包括处理过期的静默，创建快速保存
silences.Maintenance(15*time.Minute, filepath.Join(*dataDir, "silences"), stopc)

// github.com/prometheus/alertmanager/silence/silence.go
func (s *Silences) Maintenance(interval time.Duration, snapf string, stopc <-chan struct{}) {
    // ...
    // 回收处理
    if _, err := s.GC(); err != nil {
        return err
    }
 
    // 增加快照
    if size, err = s.Snapshot(f); err != nil {
        return err
    }
}

func (s *Silences) GC() (int, error) {
	// ...
	for id, sil := range s.st {
		if sil.ExpiresAt.IsZero() {
			return n, errors.New("unexpected zero expiration timestamp")
		}
		if !sil.ExpiresAt.After(now) {
			delete(s.st, id)
			delete(s.mc, sil.Silence)
			n++
		}
	}

	return n, nil
}

func (s *Silences) Snapshot(w io.Writer) (int64, error) {
    // ...
	b, err := s.st.MarshalBinary()

    // ...
	return io.Copy(w, bytes.NewReader(b))
}
```

## 4. 创建静默处理器


```go
// 创建静默处理器
silencer := silence.NewSilencer(silences, marker, logger)

// github.com/prometheus/alertmanager/silence/silence.go
// NewSilencer 创建静默处理器
func NewSilencer(s *Silences, m types.Marker, l log.Logger) *Silencer {
	return &Silencer{
		silences: s,
		marker:   m,
		logger:   l,
	}
}
```

创建静默静默判断实例

```go
// github.com/prometheus/alertmanager/notify/notify.go
ss := NewMuteStage(silencer)

// NewMuteStage 创建静默器的静默实例
func NewMuteStage(m types.Muter) *MuteStage {
	return &MuteStage{muter: m}
}

// Exec 进行静默期的静默处理
func (n *MuteStage) Exec(ctx context.Context, l log.Logger, alerts ...*types.Alert) (context.Context, []*types.Alert, error) {
	var filtered []*types.Alert
	for _, a := range alerts {
		if !n.muter.Mutes(a.Labels) {
			filtered = append(filtered, a)
		}
	}
	return ctx, filtered, nil
}
```

静默处理器更新静默状态:

```go
// 静默处理器更新静默状态
silencer.Mutes(labels)

// github.com/prometheus/alertmanager/silence/silence.go
// Mutes implements the Muter interface.
func (s *Silencer) Mutes(lset model.LabelSet) bool {
	fp := lset.Fingerprint()
	// 查找告警是否被静默，以及静默规则id
	ids, markerVersion, _ := s.marker.Silenced(fp)

	if markerVersion == s.silences.Version() {
        // 查询静默数据
		sils, _, err = s.silences.Query(
			QIDs(ids...),
			QState(types.SilenceStateActive),
		)
	} else {
		// 查询静默数据
		sils, newVersion, err = s.silences.Query(
			QState(types.SilenceStateActive),
			QMatches(lset),
		)
	}

    // 如果没有静默规则，则取消对该告警的静默状态
    if len(sils) == 0 {
		s.marker.SetSilenced(fp, newVersion)
		return false
	}

    // 如果有静默规则，还需要更新告警状态
	if idsChanged || newVersion != markerVersion {
		// Update marker only if something changed.
		s.marker.SetSilenced(fp, newVersion, ids...)
	}
	return true
}
```

## 5. 接收集群广播的静默消息

```go
// Merge 合并从集群接收的静默消息
func (s *Silences) Merge(b []byte) error {
	// 解析数据
	st, err := decodeState(bytes.NewReader(b))
	
	// 合并静默数据
	for _, e := range st {
		// ...
		s.st.merge(e, now)
	}
	return nil
}

func (s state) merge(e *pb.MeshSilence, now time.Time) bool {
	// 合并数据
	prev, ok := s[id]
	if !ok || prev.Silence.UpdatedAt.Before(e.Silence.UpdatedAt) {
		s[id] = e
		return true
	}
	return false
}
```

## 参考资料

- github.com/prometheus/alertmanager/silence/silence.go