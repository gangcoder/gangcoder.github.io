<!-- ---
title: prometheus storage
date: 2018-11-26 21:11:02
category: src, prometheus, src
--- -->

# 数据存储实现

数据存储实现，保存时序数据。

1. 创建存储实例
2. 数据添加实现
3. 更新远程存储配置
4. 开启 DB 实例
5. 查询实现

![](images/prometheus_storage.svg)


存储器使用示例：

```go
fanoutStorage = storage.NewFanout(logger, localStorage, remoteStorage)
app, err := fanoutStorage.Appender()
app.Add(s.Metric, s.T, s.V)
app.Commit()
```

## 1. 创建存储实例

```go
// 本地存储实例
type ReadyStorage struct {
	mtx sync.RWMutex
	a   *adapter
}

// 实现Storage 接口
type adapter struct {
	db              *tsdb.DB
	startTimeMargin int64
}

type appender struct {
	a tsdb.Appender
}

// Storage 远程读取和写入端点
type Storage struct {
	// 远程写入
	rws *WriteStorage
	// 远程查询
	queryables             []storage.Queryable
}

// WriteStorage 代表远程写操作
type WriteStorage struct {
	walDir            string
	queues            map[string]*QueueManager
}

// fanoutAppender  实现 Appender
type fanoutAppender struct {
	primary     Appender
	secondaries []Appender
}

// Appender 接口用于批量写入数据到存储
type Appender interface {
    // 添加数据
	Add(l labels.Labels, t int64, v float64) (uint64, error)
    // 提交数据
	Commit() error
    // 回滚
	Rollback() error
}
```

创建存储实例：

```go
//prometheus/cmd/prometheus/main.go
localStorage  = &tsdb.ReadyStorage{}
remoteStorage = remote.NewStorage(log.With(logger, "component", "remote"), localStorage.StartTime, time.Duration(cfg.RemoteFlushDeadline))
fanoutStorage = storage.NewFanout(logger, localStorage, remoteStorage)

// github.com/prometheus/prometheus/storage/fanout.go
// NewFanout 创建Storge ，用来代理底层的多个存储实例
func NewFanout(logger log.Logger, primary Storage, secondaries ...Storage) Storage {
	return &fanout{
		primary:     primary,
		secondaries: secondaries,
	}
}

// github.com/prometheus/prometheus/storage/remote/storage.go
// 创建远程存储
func NewStorage(..., walDir string, ...) *Storage {
	// 
	s := &Storage{
		localStartTimeCallback: stCallback,
	}
	s.rws = NewWriteStorage(s.logger, walDir, flushDeadline)
	return s
}

func NewWriteStorage(logger log.Logger, walDir string, flushDeadline time.Duration) *WriteStorage {
	rws := &WriteStorage{
		queues:        make(map[string]*QueueManager),
		flushDeadline: flushDeadline,
		samplesIn:     newEWMARate(ewmaWeight, shardUpdateDuration),
		walDir:        walDir,
	}
	go rws.run()
	return rws
}
```

## 2. 数据添加实现

添加数据时，先获取数据添加器，再调用添加器的 `Add` 接口。

### 2.1 Appender 数据添加器实现

```go
// fanout 添加器获取
func (f *fanout) Appender() (Appender, error) {
	// ...
	// 本地存储
	primary, err := f.primary.Appender()

	// 远程存储
	secondaries := make([]Appender, 0, len(f.secondaries))
	for _, storage := range f.secondaries {
        appender, err := storage.Appender()
        
		secondaries = append(secondaries, appender)
	}
	return &fanoutAppender{
		primary:     primary,
		secondaries: secondaries,
	}, nil
}

// Appender 本地存储添加器实现
func (s *ReadyStorage) Appender() (storage.Appender, error) {
	if x := s.get(); x != nil {
		return x.Appender()
	}
	return nil, ErrNotReady
}

// 获取 adapter
func (s *ReadyStorage) get() *adapter {
	x := s.a
	return x
}

// Appender
func (a adapter) Appender() (storage.Appender, error) {
	return appender{a: a.db.Appender()}, nil
}

// dbAppender wraps the DB's head appender and triggers compactions on commit
// if necessary.
type dbAppender struct {
	Appender
	db *DB
}

// a.db.Appender()
// Appender 创建新的添加器
func (db *DB) Appender() Appender {
	return dbAppender{db: db, Appender: db.head.Appender()}
}

// db.head.Appender()
func (h *Head) Appender() Appender {
	// ...
	return h.appender()
}

// 数据写入器
func (h *Head) appender() *headAppender {
	return &headAppender{
		head: h,
		minValidTime: max(atomic.LoadInt64(&h.minValidTime), h.MaxTime()-h.chunkRange/2),
		mint:         math.MaxInt64,
		maxt:         math.MinInt64,
		samples:      h.getAppendBuffer(),
		sampleSeries: h.getSeriesBuffer(),
	}
}
```

### 2.2 Add 数据添加实现

```go
// fanout 写入器添加实现
func (f *fanoutAppender) Add(l labels.Labels, t int64, v float64) (uint64, error) {
	ref, err := f.primary.Add(l, t, v)
	if err != nil {
		return ref, err
	}

	for _, appender := range f.secondaries {
		if _, err := appender.Add(l, t, v); err != nil {
			return 0, err
		}
	}
	return ref, nil
}

// appender
func (a appender) Add(lset labels.Labels, t int64, v float64) (uint64, error) {
    ref, err := a.a.Add(lset, t, v)
    // ...
    return ref, err
}

// headAppender
func (a *headAppender) Add(lset labels.Labels, t int64, v float64) (uint64, error) {
    // ...
    // 判断是否是新增数据
	s, created := a.head.getOrCreate(lset.Hash(), lset)
	if created {
        // 新增数据先写入 slice
		a.series = append(a.series, record.RefSeries{
			Ref:    s.ref,
			Labels: lset,
		})
	}
	return s.ref, a.AddFast(s.ref, t, v)
}

// github.com/prometheus/prometheus/tsdb/head.go
func (a *headAppender) AddFast(ref uint64, t int64, v float64) error {
	// 查找当前这条序列数据
	s := a.head.series.getByID(ref)
    
    // 设置为待提交
	s.pendingCommit = true
    
    // 添加为样本数据
	a.samples = append(a.samples, record.RefSample{
		Ref: ref,
		T:   t,
		V:   v,
    })
    // 添加为样本序列
	a.sampleSeries = append(a.sampleSeries, s)
	return nil
}
```

### 2.3 Commit 数据提交实现

```go
// github.com/prometheus/prometheus/storage/fanout.go
// 提交数据
func (f *fanoutAppender) Commit() (err error) {
	err = f.primary.Commit()

	for _, appender := range f.secondaries {
		if err == nil {
			err = appender.Commit()
		} else {
			if rollbackErr := appender.Rollback(); rollbackErr != nil {
				level.Error(f.logger).Log("msg", "Squashed rollback error on commit", "err", rollbackErr)
			}
		}
	}
	return
}

// github.com/prometheus/prometheus/storage/tsdb/tsdb.go
// primary
func (a appender) Commit() error   { return a.a.Commit() }

// github.com/prometheus/prometheus/tsdb/head.go
// headAppender
func (a *headAppender) Commit() error {
    for i, s := range a.samples {
		series = a.sampleSeries[i]
		// 提交后保存数据
		ok, chunkCreated := series.append(s.T, s.V)
		// ...
    }
    return nil
}
```

## 3. 更新远程存储配置

```go
// 更新远程存储配置
remoteStorage.ApplyConfig

// github.com/prometheus/prometheus/storage/remote/storage.go
// ApplyConfig 更新远程存储配置
func (s *Storage) ApplyConfig(conf *config.Config) error {
	// 更细储器配置
	s.rws.ApplyConfig(conf)

	for _, rrConf := range conf.RemoteReadConfigs {
		// ...

		// 更新远程请求Client
		c, err := NewClient(name, &ClientConfig{
			URL:              rrConf.URL,
			Timeout:          rrConf.RemoteTimeout,
			HTTPClientConfig: rrConf.HTTPClientConfig,
		})

		// 更新查询器
		q := QueryableClient(c)
		queryables = append(queryables, q)
	}
	s.queryables = queryables

	return nil
}
```

## 4. 开启 DB 实例

创建本地 `tsdb` 实例。

```go
// 创建tsdb 本地存储实例
db, err := tsdb.Open(
    cfg.localStoragePath,
    log.With(logger, "component", "tsdb"),
    prometheus.DefaultRegisterer,
    &cfg.tsdb,
)

// 设置本地存储实例
localStorage.Set(db, startTimeMargin)
func (s *ReadyStorage) Set(db *tsdb.DB, startTimeMargin int64) {
	s.a = &adapter{db: db, startTimeMargin: startTimeMargin}
}
```

## 5. 查询实现

```go
Querier(ctx context.Context, mint, maxt int64) (Querier, error)

// Querier 读取时间序列数据接口
type Querier interface {
	// Select 返回查询的时间序列数据
	Select(*SelectParams, ...*labels.Matcher) (SeriesSet, Warnings, error)
	// SelectSorted 返回排过序的时间序列数据
	SelectSorted(*SelectParams, ...*labels.Matcher) (SeriesSet, Warnings, error)
}

// 返回查询器
func (f *fanout) Querier(ctx context.Context, mint, maxt int64) (Querier, error) {
    queriers := make([]Querier, 0, 1+len(f.secondaries))
    
    // 本地存储查询器
    primaryQuerier, err := f.primary.Querier(ctx, mint, maxt)

	queriers = append(queriers, primaryQuerier)

	// Add secondary queriers
	for _, storage := range f.secondaries {
		querier, err := storage.Querier(ctx, mint, maxt)
		queriers = append(queriers, querier)
	}

	return NewMergeQuerier(primaryQuerier, queriers), nil
}
```

### 5.1 创建数据块查询器

本地TSDB 存储 查询实现。

```go
// Querier implements the Storage interface.
func (s *ReadyStorage) Querier(ctx context.Context, mint, maxt int64) (storage.Querier, error) {
	if x := s.get(); x != nil {
		return x.Querier(ctx, mint, maxt)
	}
	return nil, ErrNotReady
}

type querier struct {
	q tsdb.Querier
}

// github.com/prometheus/prometheus/storage/tsdb/tsdb.go
func (a adapter) Querier(_ context.Context, mint, maxt int64) (storage.Querier, error) {
	q, err := a.db.Querier(mint, maxt)
	// ...
	return querier{q: q}, nil
}

// a.db.Querier(mint, maxt)
// github.com/prometheus/prometheus/tsdb/db.go
// Querier 时间序列数据查询器
// 查找需要查询的数据块
func (db *DB) Querier(mint, maxt int64) (Querier, error) {
	var blocks []BlockReader
	var blockMetas []BlockMeta

	// 查询匹配的数据库
	for _, b := range db.blocks {
		if b.OverlapsClosedInterval(mint, maxt) {
			blocks = append(blocks, b)
			blockMetas = append(blockMetas, b.Meta())
		}
	}

	blockQueriers := make([]Querier, 0, len(blocks))
	for _, b := range blocks {
		q, err := NewBlockQuerier(b, mint, maxt)
		if err == nil {
			blockQueriers = append(blockQueriers, q)
			continue
		}
		// ...
	}

    // ...
	return &querier{
		blocks: blockQueriers,
	}, nil
}

// github.com/prometheus/prometheus/tsdb/querier.go
// NewBlockQuerier 创建数据块的查询器
func NewBlockQuerier(b BlockReader, mint, maxt int64) (Querier, error) {
	indexr, err := b.Index()
    
	chunkr, err := b.Chunks()
	
	tombsr, err := b.Tombstones()
	// ...
	return &blockQuerier{
		mint:       mint,
		maxt:       maxt,
		index:      indexr,
		chunks:     chunkr,
		tombstones: tombsr,
	}, nil
}
```

### 5.2 数据查询实现

```go
// github.com/prometheus/prometheus/storage/tsdb/tsdb.go
func (q querier) Select(_ *storage.SelectParams, ms ...*labels.Matcher) (storage.SeriesSet, storage.Warnings, error) {
	set, err := q.q.Select(ms...)
    // ...
	return seriesSet{set: set}, nil, nil
}

// github.com/prometheus/prometheus/tsdb/db.go
func (q *querier) Select(ms ...*labels.Matcher) (SeriesSet, error) {
    // 如果数据块不止一个，需要排序查询结果
    if len(q.blocks) != 1 {
		return q.SelectSorted(ms...)
	}
	// 
	return q.blocks[0].Select(ms...)
}

// 数据块查询
func (q *blockQuerier) Select(ms ...*labels.Matcher) (SeriesSet, error) {
    // 查询时序数据
    base, err := LookupChunkSeries(q.index, q.tombstones, ms...)

	return &blockSeriesSet{
		set: &populatedChunkSeries{
			set:    base,
			chunks: q.chunks,
			mint:   q.mint,
			maxt:   q.maxt,
		},
	}, nil
}
```

## 参考资料

- github.com/prometheus/prometheus/cmd/prometheus/main.go
- github.com/prometheus/prometheus/scrape/manager.go
