<!-- ---
title: prometheus discovery
date: 2018-11-26 17:31:18
category: src, prometheus, src
--- -->

# 服务发现实现

从服务发现服务查询最新的目标地址，Prometheus 中使用服务发现功能的是：查询抓取目标客户端机器地址和查询告警中心地址。

服务发现逻辑：

1. 创建服务发现实例：创建Manager 实例
2. 载入服务发现配置
3. 服务发现执行逻辑
4. 获取服务发现结果：返回包含结果的 chan
5. 服务发现提供者逻辑

![](images/prometheus_discovery.svg)

服务发现代码使用示例：

```go
discoveryManagerNotify = discovery.NewManager(ctxNotify, logger, discovery.Name("notify"))
discoveryManagerNotify.ApplyConfig(c)
discoveryManagerNotify.Run()
discoveryManagerNotify.SyncCh()
```

## 1. 创建服务发现实例

初始化Discovery 实例。

```go
// 创建服务发现实例
//prometheus/cmd/prometheus/main.go
discoveryManagerNotify  = discovery.NewManager(ctxNotify, log.With(logger, "component", "discovery manager notify"), discovery.Name("notify"))

// 目标机器实例
type Group struct {
	// Targets 目标信息
	Targets []model.LabelSet
	// Labels 目标标签
    Labels model.LabelSet

	// Source
	Source string
}

// provider 服务发现提供者
type provider struct {
	name   string
	d      Discoverer
	subs   []string
	config interface{}
}

// Manager 服务发现结构体
type Manager struct {
    logger         log.Logger
    name           string
    mtx            sync.RWMutex
    ctx            context.Context
    discoverCancel []context.CancelFunc

    targets map[poolKey]map[string]*targetgroup.Group // 保存发现的目标机器，poolKey 用与区分不同的发现服务
    // 服务发现所使用的发现服务
    providers []*provider
    // 保存所有发现的目标地址， 并通过这个chan 暴露地址给使用者
    syncCh chan map[string][]*targetgroup.Group

    // 更新周期
    updatert time.Duration

    // 触发manager 处理更新
    triggerSend chan struct{}
}

// NewManager 创建服务发现实例
func NewManager(ctx context.Context, logger log.Logger, options ...func(*Manager)) *Manager {
    mgr := &Manager{
        logger:         logger,
        syncCh:         make(chan map[string][]*targetgroup.Group),
        targets:        make(map[poolKey]map[string]*targetgroup.Group),
        discoverCancel: []context.CancelFunc{},
        ctx:            ctx,
        updatert:       5 * time.Second,
        triggerSend:    make(chan struct{}, 1),
    }
    for _, option := range options {
        option(mgr)
    }
    return mgr
}
```


## 2. 载入服务发现配置

载入服务发现需要的配置，在外部配置更新后也会调用这个函数更新配置。

重载配置的过程中，会获取一次服务发现目标数据，并且将目标更新到 `m.targets` 中：

1. 更新配置，创建和注册发现服务提供者
2. 运行注册的发现服务提供者
3. 发现服务会将获取到的target 写入channel
4. Manager 再从 `channel` 中获取targets 写入 `m.targets`

```go
//server/main.go
// return discoveryManagerNotify.ApplyConfig(c)

// ApplyConfig 移除已有的服务提供者，并且基于配置创建新的服务提供者
func (m *Manager) ApplyConfig(cfg map[string]sd_config.ServiceDiscoveryConfig) error {
    // 停止服务发现逻辑
    m.cancelDiscoverers()
    m.targets = make(map[poolKey]map[string]*targetgroup.Group)
	m.providers = nil
	m.discoverCancel = nil

    // 基于配置创建新的发现服务提供者
    // 发现服务会注册到 m.providers 中
    for name, scfg := range cfg {
        // 重新注册服务发现提供者
        m.registerProviders(scfg, name)
    }

    // 注册完发现服务，开始运行发现服务
    for _, prov := range m.providers {
        //运行提供者进程进行查询
        m.startProvider(m.ctx, prov)
    }

    return nil
}

// 停止服务发现提供者逻辑
func (m *Manager) cancelDiscoverers() {
	for _, c := range m.discoverCancel {
		c()
	}
}

// 注册发现服务提供者
func (m *Manager) registerProviders(cfg sd_config.ServiceDiscoveryConfig, setName string) {
    add := func(cfg interface{}, newDiscoverer func() (Discoverer, error)) {
        // 检查发现服务是否存在，如果存在直接返回
        t := reflect.TypeOf(cfg).String()
        for _, p := range m.providers {
            if reflect.DeepEqual(cfg, p.config) {
                p.subs = append(p.subs, setName)
                return
            }
        }

        // 创建新的发现服务操作实例
        d, err := newDiscoverer()

        // 基于发现服务和配置，创建新的发现服务
        provider := provider{
            name:   fmt.Sprintf("%s/%d", t, len(m.providers)),
            d:      d,
            config: cfg,
            subs:   []string{setName},
        }

        // 将发现服务实例注册到 m.providers
        m.providers = append(m.providers, &provider)
    }

    // 依次遍历DNS, File, Consul, MarathonSD 配置信息，如果有配置相关服务发现逻辑，就创建发现服务实例
    for _, c := range cfg.DNSSDConfigs {
        add(c, func() (Discoverer, error) {
            return dns.NewDiscovery(*c, log.With(m.logger, "discovery", "dns")), nil
        })
    }
    
    for _, c := range cfg.FileSDConfigs {
        add(c, func() (Discoverer, error) {
            return file.NewDiscovery(c, log.With(m.logger, "discovery", "file")), nil
        })
    }
    
    for _, c := range cfg.ConsulSDConfigs {
        add(c, func() (Discoverer, error) {
            return consul.NewDiscovery(c, log.With(m.logger, "discovery", "consul"))
        })
    }
    // ...
    if len(cfg.StaticConfigs) > 0 {
        add(setName, func() (Discoverer, error) {
            return &StaticProvider{cfg.StaticConfigs}, nil
        })
    }
}

//运行提供者进程进行查询
func (m *Manager) startProvider(ctx context.Context, p *provider) {
    ctx, cancel := context.WithCancel(ctx)
    updates := make(chan []*targetgroup.Group)

    m.discoverCancel = append(m.discoverCancel, cancel)

    //运行服务发现提供者查询，获取查询结果
    go p.d.Run(ctx, updates) //运行服务发现提供者
    go m.updater(ctx, p, updates) //更新服务发现提供者提供的数据
}

// 更新发现服务发现的目标数据
func (m *Manager) updater(ctx context.Context, p *provider, updates chan []*targetgroup.Group) {
    for {
        select {
        case <-ctx.Done():
            return
        case tgs, ok := <-updates:
            // 服务发现提供者发现的目标数据
            for _, s := range p.subs {
                m.updateGroup(poolKey{setName: s, provider: p.name}, tgs)
            }

            select {
            case m.triggerSend <- struct{}{}:
            default:
            }
        }
    }
}

// 更新服务发现获取到的目标数据，将数据写入到 m.targets 中
// 注意这里的poolKey 有p.subs 名称和p.name 组成
func (m *Manager) updateGroup(poolKey poolKey, tgs []*targetgroup.Group) {
    for _, tg := range tgs {
        m.targets[poolKey][tg.Source] = tg
    }
}
```

## 3. 服务发现执行逻辑

Run 处理逻辑：主要是将 `m.targets` 中的target 分组后写入channel

1. 定时获取所有的target
2. 将target 写入syncCh 这个channel 中

```go
// cmd/main.go 主程序中执行Run
err := discoveryManagerNotify.Run()

// Run 开启后端处理
// 定时获取所有的target
func (m *Manager) Run() error {
    go m.sender()
    // ...
    return nil
}

// 定时获取所有的target
// 将target 写入syncCh 这个channel 中
func (m *Manager) sender() {
    ticker := time.NewTicker(m.updatert)
    defer ticker.Stop()

    for {
        select {
            // 停止运行
        case <-m.ctx.Done():
            return
        case <-ticker.C: // 每隔一段时间在处理发现服务的更新，避免更新太过频繁
            select {
            case <-m.triggerSend: // 有更新时，发现服务会往chan 中写入信号
                select {
                case m.syncCh <- m.allGroups(): // 如果m.syncCh 被消费，就往 syncCh 中写入最新的目标机器列表数据
                default:
                    // ...
                }
            default:
            }
        }
    }
}

// allGroups 获取所有发现的目标地址
// 按照targetgroup 名字对target 进行分组，然后返回
func (m *Manager) allGroups() map[string][]*targetgroup.Group {
    // allGroups
    // 按照pkey.setName 重新组织数据
    tSets := map[string][]*targetgroup.Group{}
    for pkey, tsets := range m.targets {
        var n int
        for _, tg := range tsets {
            tSets[pkey.setName] = append(tSets[pkey.setName], tg)
            n += len(tg.Targets)
        }
    }
    return tSets
}
```

## 4. 获取服务发现结果

服务发现使用方获取服务发现结果，取出 `Discovery Manager` 获取到的target，这些target 由定时任务写入syncCh。

```go
// cmd/main.go 中调用
notifier.Run(discoveryManagerNotify.SyncCh())

// SyncCh 通过一个只读的 channel 返回所有的目标地址更新数据.
func (m *Manager) SyncCh() <-chan map[string][]*targetgroup.Group {
    return m.syncCh
}
```

## 5. 服务发现提供者逻辑

具体的服务发现提供者有基于 DNS, File, Consul, zookeeper 等的实现。

服务发现提供者需要实现2 个逻辑：

1. 创建服务发现提供者实例
2. 注册服务发现提供者
3. 运行服务发现逻辑，获取最新发现结果

这两个逻辑分别对应 `discovery/manager.go` 中的服务提供者注册和运行服务发现调用。

```go
// prometheus/prometheus/discovery/manager.go

// Discoverer 服务发现提供者接口
type Discoverer interface {
	// Run 通过up 这个channel，服务发现提供者可以返回最新的目标信息
	Run(ctx context.Context, up chan<- []*targetgroup.Group)
}

// 1. 创建服务发现提供者实例
// 2. 注册服务发现提供者
func (m *Manager) registerProviders(cfg sd_config.ServiceDiscoveryConfig, setName string) {
    add := func(cfg interface{}, newDiscoverer func() (Discoverer, error)) {
        // 创建服务发现提供者
		d, err := newDiscoverer()
        
		provider := provider{
			name:   fmt.Sprintf("%s/%d", t, len(m.providers)),
			d:      d,
			config: cfg,
			subs:   []string{setName},
        }
        // 2. 注册服务发现提供者
		m.providers = append(m.providers, &provider)
	}
    // ...
    // 注册基于文件服务发现服务
	for _, c := range cfg.FileSDConfigs {
        // 第二个参数是一个函数，这个函数会返回一个 Discoverer 类型的服务发现对象
		add(c, func() (Discoverer, error) {
			return file.NewDiscovery(c, log.With(m.logger, "discovery", "file")), nil
		})
	}
}

// 3. 运行服务发现逻辑，获取最新发现结果
go p.d.Run(ctx, updates) //获取服务发现查询结果
```

### 5.1 file 服务发现实现

```go
// Discovery 基于文件的服务发现实例
type Discovery struct {
	paths      []string
	watcher    *fsnotify.Watcher // 监控文件修改
	interval   time.Duration
	timestamps map[string]float64

	// lastRefresh 文件最后一次解析出来的目标地址数量，用于在配置文件失效或者被删除时平滑删除
	lastRefresh map[string]int
}

// NewDiscovery returns a new file discovery for the given paths.
func NewDiscovery(conf *SDConfig, logger log.Logger) *Discovery {
    // 创建一个服务发现实例
	disc := &Discovery{
		paths:      conf.Files,
		interval:   time.Duration(conf.RefreshInterval),
		timestamps: make(map[string]float64),
		logger:     logger,
    }
	return disc
}

// Run 实现 Discoverer 接口
func (d *Discovery) Run(ctx context.Context, ch chan<- []*targetgroup.Group) {
    watcher, err := fsnotify.NewWatcher()
    // ...
	d.watcher = watcher
	defer d.stop()

    // 初始化时获取文件配置
	d.refresh(ctx, ch)

	ticker := time.NewTicker(d.interval)
	defer ticker.Stop()
	for {
		select {
        // 停止运行
		case <-ctx.Done():
			return
		case event := <-d.watcher.Events:
			// 当检测到文件有变化时，获取文件配置数据
			d.refresh(ctx, ch)

		case <-ticker.C:
			// 定时检查文件配置数据
			d.refresh(ctx, ch)
		}
	}
}

// refresh 读取所有配置文件，解析配置项
func (d *Discovery) refresh(ctx context.Context, ch chan<- []*targetgroup.Group) {
	// ...
    // 遍历所有的配置文件
	for _, p := range d.listFiles() {
        // 读取配置项
		tgroups, err := d.readFile(p)
		if err != nil {
			// ...
			continue
        }
        // 配置文件中的目标机器信息写入Discover Manager 传入的chan 中
		select {
		case ch <- tgroups:
		case <-ctx.Done():
			return
		}

		ref[p] = len(tgroups)
    }
    
    // ...
	d.lastRefresh = ref

    // 更新需要监控文件变化的文件列表
	d.watchFiles()
}
```

### 5.2 consul 服务发现实现

```go
// Discovery 基于consule 的服务发现提供者，从Consul 服务端获取目标信息
type Discovery struct {
	client           *consul.Client
	clientDatacenter string
	tagSeparator     string
	watchedServices  []string // Set of services which will be discovered.
	watchedTag       string   // A tag used to filter instances of a service.
	watchedNodeMeta  map[string]string
	allowStale       bool
	refreshInterval  time.Duration
	finalizer        func()
	logger           log.Logger
}

// NewDiscovery 创建一个consul 服务发现客户端实例
func NewDiscovery(conf *SDConfig, logger log.Logger) (*Discovery, error) {
	// consul 客户端和服务端使用http 协议进行通信
	wrapper := &http.Client{
		Transport: transport,
		Timeout:   35 * time.Second,
	}

	// 创建consul 客户端实例
	client, err := consul.NewClient(clientConf)
    // ...
    
    // 创建一个基于consul 的服务发现实例
	cd := &Discovery{
		client:           client,
		tagSeparator:     conf.TagSeparator,
		watchedServices:  conf.Services,
		watchedTags:      conf.ServiceTags,
		watchedNodeMeta:  conf.NodeMeta,
		allowStale:       conf.AllowStale,
		refreshInterval:  time.Duration(conf.RefreshInterval),
		clientDatacenter: conf.Datacenter,
		finalizer:        transport.CloseIdleConnections,
		logger:           logger,
	}
	return cd, nil
}

// Run 从consul 服务层获取数据信息
func (d *Discovery) Run(ctx context.Context, ch chan<- []*targetgroup.Group) {
    // 第一次初始化时，不知道需要检测哪些服务
    // 需要开启一个goroutine 定时获取需要监控的最全服务列表
	if len(d.watchedServices) == 0 || d.watchedTag != "" {
		// We need to watch the catalog.
		ticker := time.NewTicker(d.refreshInterval)

		for {
			select {
			case <-ctx.Done():
				ticker.Stop()
				return
            default:
                // 获取服务列表
				d.watchServices(ctx, ch, &lastIndex, services)
				<-ticker.C
			}
		}

	} else {
        // 当知道需要监控的服务列表后，针对单个服务开启监控
        // 获取的数据写入 ch 这个参数channel 中
		for _, name := range d.watchedServices {
			d.watchService(ctx, ch, name)
		}
		<-ctx.Done()
	}
}

// 当不知道有哪些服务时，用来查询所有的服务名称
func (d *Discovery) watchServices(ctx context.Context, ch chan<- []*targetgroup.Group, lastIndex *uint64, services map[string]func()) error {
	// 查询consul 的所有目录
    catalog := d.client.Catalog()
	opts := &consul.QueryOptions{
		WaitIndex:  *lastIndex,
		WaitTime:   watchTimeout,
		AllowStale: d.allowStale,
		NodeMeta:   d.watchedNodeMeta,
	}
    srvs, meta, err := catalog.Services(opts.WithContext(ctx))
    
    // 检查查询到的服务是否需要监控
    // 对新查询到的服务开启信息获取
	for name := range srvs {
		// 判断是否需要监测
		if !d.shouldWatch(name, srvs[name]) {
			continue
		}
		
		// 对于新发现的服务，需要立马去查询服务获取信息
		d.watchService(wctx, ch, name)
		services[name] = cancel
	}

	return nil
}


// consulService contains data belonging to the same service.
type consulService struct {
	name         string
	labels       model.LabelSet
	discovery    *Discovery
	client       *consul.Client
}

// 开始监测某个服务
func (d *Discovery) watchService(ctx context.Context, ch chan<- []*targetgroup.Group, name string) {
	srv := &consulService{
		discovery: d,
		client:    d.client,
		name:      name,
		tag:       d.watchedTag,
		labels: model.LabelSet{
			serviceLabel:    model.LabelValue(name),
			datacenterLabel: model.LabelValue(d.clientDatacenter),
		},
		tagSeparator: d.tagSeparator,
		logger:       d.logger,
	}

    // goroutine 定时抓取最新目标客户端数据
	go func() {
		ticker := time.NewTicker(d.refreshInterval)
		var lastIndex uint64
		catalog := srv.client.Catalog()
		for {
			select {
			case <-ctx.Done():
				ticker.Stop()
				return
            default:
                // 持续监控服务发现的某个服务，并且定时获取新的信息
				srv.watch(ctx, ch, catalog, &lastIndex)
				<-ticker.C
			}
		}
	}()
}

// 更新一个服务的信息
func (srv *consulService) watch(ctx context.Context, ch chan<- []*targetgroup.Group, catalog *consul.Catalog, lastIndex *uint64) error {
	// 获取服务发现服务中某个服务的最新信息
	opts := &consul.QueryOptions{
		WaitIndex:  *lastIndex,
		WaitTime:   watchTimeout,
		AllowStale: srv.discovery.allowStale,
		NodeMeta:   srv.discovery.watchedNodeMeta,
	}
	nodes, meta, err := catalog.ServiceMultipleTags(srv.name, srv.tags, opts.WithContext(ctx))
    
    // 将抓取的最新节点信息格式化为需要的格式
	for _, node := range nodes {
        // 将节点信息格式化为内部节点信息
		labels := model.LabelSet{
			model.AddressLabel:  model.LabelValue(addr),
			addressLabel:        model.LabelValue(node.Address),
			nodeLabel:           model.LabelValue(node.Node),
			tagsLabel:           model.LabelValue(tags),
			serviceAddressLabel: model.LabelValue(node.ServiceAddress),
			servicePortLabel:    model.LabelValue(strconv.Itoa(node.ServicePort)),
			serviceIDLabel:      model.LabelValue(node.ServiceID),
        }
        
		tgroup.Targets = append(tgroup.Targets, labels)
	}

    // 将信息写入channel
	select {
	case <-ctx.Done():
	case ch <- []*targetgroup.Group{&tgroup}:
	}
}
```


## 参考资料

- github.com/prometheus/prometheus/cmd/prometheus/main.go
- github.com/prometheus/prometheus/discovery/manager.go
