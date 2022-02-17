<!-- ---
title: prometheus scrape
date: 2018-11-19 11:06:22
category: src, prometheus, src
--- -->

# Scrape 抓取服务实现

scrape 负责从目标客户端机器抓取Metric 指标数据，并且将指标数据落地。

scrape 实现抓取Metric 功能，主要逻辑如下：

1. 创建scrape 实例
2. 载入scrape 配置
3. 运行scrape 抓取逻辑，包括从scrape 服务发现获取目标机器地址，开启goroutine 循环抓取指标数据
4. 抓取数据，存入底层存储，并且上报scrape 指标数据和目标机器在线状态


![](images/prometheus_scrape.svg)


抓取服务代码使用示例：

```go
// main.go
// 创建scrape 管理器实例
scrapeManager = scrape.NewManager(log.With(logger, "component", "scrape manager"), fanoutStorage)

// 载入scrape 配置
scrapeManager.ApplyConfig

// 创建scrape 服务发现实例，用于发现目标客户端机器地址
discoveryManagerScrape  = discovery.NewManager(ctxScrape, log.With(logger, "component", "discovery manager scrape"), discovery.Name("scrape"))

//运行scrape 管理器实例主逻辑
err := scrapeManager.Run(discoveryManagerScrape.SyncCh())
```

## 1. 创建scrape 实例

`main.go` 中创建 scrape Manager 实例。

```go
// github.com/prometheus/prometheus/cmd/prometheus/main.go
// fanoutStorage 用于实现数据存储逻辑
localStorage  = &tsdb.ReadyStorage{}
remoteStorage = remote.NewStorage(log.With(logger, "component", "remote"), prometheus.DefaultRegisterer, localStorage.StartTime, cfg.localStoragePath, time.Duration(cfg.RemoteFlushDeadline))
fanoutStorage = storage.NewFanout(logger, localStorage, remoteStorage)

// scrape 服务发现实例
discoveryManagerScrape  = discovery.NewManager(ctxScrape, log.With(logger, "component", "discovery manager scrape"), discovery.Name("scrape"))

// scrape Manager 实例
scrapeManager = scrape.NewManager(log.With(logger, "component", "scrape manager"), fanoutStorage)
```

scrape 管理器结构如下：

```go
// Manager 管理着抓取客户端机器的处理池和抓取生命周期
type Manager struct {
	append    Appendable // 数据存储实例对象
	
	mtxScrape     sync.Mutex // Guards the fields below.
	scrapeConfigs map[string]*config.ScrapeConfig // 抓取配置
	scrapePools   map[string]*scrapePool // scrapePools 的map key 和 scrapeConfigs 的相同，都是用的目标机器的 jobName
	targetSets    map[string][]*targetgroup.Group // 抓取目标

	triggerReload chan struct{}
}

// scrape/scrape.go
// scrapePool 用于对一系列目标的抓取
type scrapePool struct {
	appendable Appendable
	
	config *config.ScrapeConfig
	client *http.Client
	// 抓取目标机器
	activeTargets  map[uint64]*Target
	droppedTargets []*Target
	loops          map[uint64]loop
	cancel         context.CancelFunc

	// Constructor 创建新的抓取循环
	newLoop func(*Target, scraper, int, bool, []*config.RelabelConfig) loop
}

// 实现了loop 接口
type scrapeLoop struct {
	scraper        scraper
	l              log.Logger
	cache          *scrapeCache
	lastScrapeSize int
	buffers        *pool.Pool

	appender            func() storage.Appender
	sampleMutator       labelsMutator
	reportSampleMutator labelsMutator

	ctx       context.Context
	cancel    func()
	stopped   chan struct{}
}

// 新建实例
func NewManager(logger log.Logger, app Appendable) *Manager {
	return &Manager{
		append:        app,
		logger:        logger,
		scrapeConfigs: make(map[string]*config.ScrapeConfig),
		scrapePools:   make(map[string]*scrapePool),
		graceShut:     make(chan struct{}),
		triggerReload: make(chan struct{}, 1),
	}
}
```

## 2. 载入scrape 配置

```go
// main.go 中会解析配置
// scrapeManager.ApplyConfig

// Config 配置文件
type Config struct {
	// scrape 配置
	ScrapeConfigs  []*ScrapeConfig `yaml:"scrape_configs,omitempty"`
}

// ApplyConfig 设置scrape manager 的服务发现和抓取任务配置
func (m *Manager) ApplyConfig(cfg *config.Config) error {
	// ... 调整抓取配置任务
	c := make(map[string]*config.ScrapeConfig)
	for _, scfg := range cfg.ScrapeConfigs {
		c[scfg.JobName] = scfg
	}

	// 设置manager 的抓取配置
	m.scrapeConfigs = c

	// 配置调整后需要重新加载pool，并且清理可能不需要抓取的目标
	for name, sp := range m.scrapePools {
		// pools 不在新的配置中时，需要停止抓取
		if cfg, ok := m.scrapeConfigs[name]; !ok {
			sp.stop()
			delete(m.scrapePools, name)
		} else if !reflect.DeepEqual(sp.config, cfg) {
			sp.reload(cfg)
		}
	}

	return nil
}

// reload 重新启动对目标机器的抓取任务
func (sp *scrapePool) reload(cfg *config.ScrapeConfig) {
	// ...
	client, err := config_util.NewClientFromConfig(cfg.HTTPClientConfig, cfg.JobName, false)
	
	// 使用新配置创建新的Loop 并且关闭旧的Loop
	for fp, oldLoop := range sp.loops {
		var (
			t       = sp.activeTargets[fp]
			s       = &targetScraper{Target: t, client: sp.client, timeout: timeout}
			newLoop = sp.newLoop(scrapeLoopOptions{
				target: t,
				scraper: s,
			})
		)
		wg.Add(1)

		go func(oldLoop, newLoop loop) {
			oldLoop.stop()
			wg.Done()

			go newLoop.run(interval, timeout, nil)
		}(oldLoop, newLoop)

		sp.loops[fp] = newLoop
	}

	wg.Wait()
}

type loop interface {
	run(interval, timeout time.Duration, errc chan<- error)
	stop()
}

// 创建新的抓取循环
sp.newLoop = func(opts scrapeLoopOptions) loop {
	// ...
	return newScrapeLoop(ctx, opts.scraper, ... )
}

func newScrapeLoop(ctx context.Context, sc scraper, ...) *scrapeLoop {
	// ...	
	sl := &scrapeLoop{
		scraper: sc,
	}

	return sl
}
```

## 3. 运行scrape 抓取逻辑

运行抓取客户端指标的主逻辑。

```go
// 运行scrape
func (m *Manager) Run(tsets <-chan map[string][]*targetgroup.Group) error {
	// 后端执行重载更新
	go m.reloader()
	for {
		select {
		case ts := <-tsets:
			// 更新targets 列表
			m.updateTsets(ts)

			// 触发重载
			select {
			case m.triggerReload <- struct{}{}:
			default:
			}

		case <-m.graceShut:
			return nil
		}
	}
}

// 更新m.targetSets 目标客户端地址列表
func (m *Manager) updateTsets(tsets map[string][]*targetgroup.Group) {
	m.mtxScrape.Lock()
	m.targetSets = tsets
	m.mtxScrape.Unlock()
}

// 重载处理
func (m *Manager) reloader() {
    m.reload()
}

// 从m.targetSets 中取出所有目标地址
// 处理每一个 scrape
func (m *Manager) reload() {
	// 取出目标地址
	for setName, groups := range m.targetSets {
		// 检查抓取pool 是否存在，不存在就新建
		if _, ok := m.scrapePools[setName]; !ok {
			// 新建pool 前，检查抓取配置是否存在，不存在就退出
			scrapeConfig, ok := m.scrapeConfigs[setName]
			if !ok {
				continue
			}
			sp, err := newScrapePool(scrapeConfig, m.append, m.jitterSeed, log.With(m.logger, "scrape_pool", setName))

			m.scrapePools[setName] = sp
		}

		wg.Add(1)
		// 开启并发抓取
		go func(sp *scrapePool, groups []*targetgroup.Group) {
			sp.Sync(groups)
			wg.Done()
		}(m.scrapePools[setName], groups)

	}
	m.mtxScrape.Unlock()
	wg.Wait()
}

func newScrapePool(cfg *config.ScrapeConfig, app Appendable, jitterSeed uint64, logger log.Logger) (*scrapePool, error) {
	// ...
	sp := &scrapePool{
		cancel:        cancel,
		appendable:    app,
		config:        cfg,
		client:        client,
		activeTargets: map[uint64]*Target{},
		loops:         map[uint64]loop{},
		logger:        logger,
	}
	sp.newLoop = func(opts scrapeLoopOptions) loop {
		// ...
		return newScrapeLoop(
			ctx,
			opts.scraper,
			...
		)
	}

	return sp, nil
}

// 异步处理抓取客户端数据任务
func (sp *scrapePool) Sync(tgs []*targetgroup.Group) {
	var all []*Target
	sp.mtx.Lock()
	sp.droppedTargets = []*Target{}
	// 根据targetgroup 目标机器的标签，区分那些是可以抓取的目标地址，那些客户端地址已经过期
	for _, tg := range tgs {
		targets, err := targetsFromGroup(tg, sp.config)
		if err != nil {
			level.Error(sp.logger).Log("msg", "creating targets failed", "err", err)
			continue
		}
		for _, t := range targets {
			// 有效的抓取地址
			if t.Labels().Len() > 0 {
				all = append(all, t)
			} else if t.DiscoveredLabels().Len() > 0 {
				// 失效的抓取地址
				sp.droppedTargets = append(sp.droppedTargets, t)
			}
		}
	}
	sp.mtx.Unlock()
	// 开启抓取
	sp.sync(all)
}

// 去重，开启loop，并且关闭失效的targets
func (sp *scrapePool) sync(targets []*Target) {
	// ...
	for _, t := range targets {
		// 获取每个抓取目标的hash 值
		t := t
		hash := t.hash()
		uniqueTargets[hash] = struct{}{}

		// 检查目标是否已经存在在 activeTargets 中
		if _, ok := sp.activeTargets[hash]; !ok {
			// 如果不存在就创建新的loop
			s := &targetScraper{Target: t, client: sp.client, timeout: timeout}
			l := sp.newLoop(scrapeLoopOptions{
				target:          t,
				scraper:         s,
				limit:           limit,
				honorLabels:     honorLabels,
				honorTimestamps: honorTimestamps,
				mrc:             mrc,
			})

			sp.activeTargets[hash] = t
			sp.loops[hash] = l

			go l.run(interval, timeout, nil)
		} else {
			// 如果存在，需要更新下监控信息
			sp.activeTargets[hash].SetDiscoveredLabels(t.DiscoveredLabels())
		}
	}

	// ...
	// 检查activeTargets，如果不在本次更新的目标机器中，则需要关掉loop
	for hash := range sp.activeTargets {
		if _, ok := uniqueTargets[hash]; !ok {
			go func(l loop) {
				l.stop()
			}(sp.loops[hash])

			delete(sp.loops, hash)
			delete(sp.activeTargets, hash)
		}
	}
}
```

## 4. scrapeLoop 抓取任务

负责从某一个指定的机器抓取指标数据。

### 4.1 执行抓取

从目标机器抓取数据，并且将数据存入底层存储中，这里为了提高性能，用到了 `bytes.NewsBuffer`。

```go
// 执行loop run 函数
func (sl *scrapeLoop) run(interval, timeout time.Duration, errc chan<- error) {
	ticker := time.NewTicker(interval)
	defer ticker.Stop()

	// 主循环
	for {
		buf.Reset()
		
		// 从buffers 池中获取bytes
		b := sl.buffers.Get(sl.lastScrapeSize).([]byte)
		buf := bytes.NewBuffer(b)

		// 抓取客户端指标数据
		contentType, scrapeErr := sl.scraper.scrape(scrapeCtx, buf)
		cancel()

		b = buf.Bytes()

		// 将指标数据写入底层存储
		total, added, seriesAdded, appErr := sl.append(b, contentType, start)
		
		// 将bytes 放回buffers 池
		sl.buffers.Put(b)
		
		// 上报抓取监控指标，用于监控scrape 操作本身
		if err := sl.report(start, time.Since(start), total, added, scrapeErr); err != nil {
			level.Warn(sl.l).Log("msg", "appending scrape report failed", "err", err)
		}
	}

	close(sl.stopped)
	sl.endOfRunStaleness(last, ticker, interval)
}
```

### 4.2 抓取数据

targetScraper 实现了从目标发起http 请求进行抓取的逻辑

```go
// scraper 接口定义，用于取回样本数据，并且接受一个目标机器是否健康的状态报告
type scraper interface {
	scrape(ctx context.Context, w io.Writer) (string, error)
	report(start time.Time, dur time.Duration, err error)
	offset(interval time.Duration) time.Duration
}

// targetScraper 实现针对客户端目标机器的 scrape 接口
type targetScraper struct {
	*Target
	client  *http.Client
}

// 抓取数据
func (s *targetScraper) scrape(ctx context.Context, w io.Writer) (string, error) {
	// 发送请求
	resp, err := s.client.Do(s.req.WithContext(ctx))
	defer func() {
		io.Copy(ioutil.Discard, resp.Body)
		resp.Body.Close()
	}()

	// 解压缩读取的数据
	s.buf = bufio.NewReader(resp.Body)
	s.gzipr, err = gzip.NewReader(s.buf)

	_, err = io.Copy(w, s.gzipr)
	s.gzipr.Close()
	
	return resp.Header.Get("Content-Type"), nil
}
```

### 4.3 上报抓取情况

report 函数实现上报抓取情况的功能，包括抓取目标机器的在线状态，和本次抓取的监控指标数据

```go
func (sl *scrapeLoop) report(start time.Time, duration time.Duration, scraped, appended int, err error) error {
	// 上报抓取目标健康状态
	sl.scraper.report(start, duration, err)

	ts := timestamp.FromTime(start)

	// 抓取是否正常
	var health float64
	if err == nil {
		health = 1
	}

	// 底层存储
	app := sl.appender()

	// 目标客户端是否在线
	if err := sl.addReportSample(app, scrapeHealthMetricName, ts, health); err != nil {
		app.Rollback()
		return err
	}

	// 抓取用时
	if err := sl.addReportSample(app, scrapeDurationMetricName, ts, duration.Seconds()); err != nil {
		app.Rollback()
		return err
	}
	// ...
	return app.Commit()
}
```


## 参考资料

- github.com/prometheus/prometheus/cmd/prometheus/main.go
- github.com/prometheus/prometheus/scrape/manager.go
- github.com/prometheus/prometheus/scrape/scrape.go
- github.com/prometheus/prometheus/scrape/target.go
