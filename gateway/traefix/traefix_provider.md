<!-- ---
title: Traefix Provider 实现
date: 2020-07-11 16:20:12
category: showcode, gateway, traefix
--- -->

# Traefix Provider 实现

配置提供器，用于提供最新的配置变更。配置通过 `chan<- dynamic.Message` 返回。

![](images/traefix_provider.svg)

主要数据结构：

```go
// 配置提供器的聚合处理
// 聚合多种配置提供器的配置
type ProviderAggregator struct {
    fileProvider *file.Provider
    providers    []provider.Provider
}

// 提供器接口
type Provider interface {
    // Provide 函数通过channel 将变动后的配置提供给traefix
    Provide(configurationChan chan<- dynamic.Message, pool *safe.Pool) error
    Init() error
}

// 文件配置提供器
type Provider struct {
    Directory                 string
    Watch                     bool  
    Filename                  string
    DebugLogGeneratedTemplate bool  
}

// consul catalog 配置提供器
type Provider struct {
    Constraints       string
    Endpoint          *EndpointConfig
    Prefix            string
    client         *api.Client
    defaultRuleTpl *template.Template
}
```

主要代码：

```go
// 创建配置提供器实例
providerAggregator := aggregator.NewProviderAggregator(*staticConfiguration.Providers)

// 添加内部配置提供器
err := providerAggregator.AddProvider(traefik.New(*staticConfiguration))

// 获取配置提供器的配置
currentProvider.Provide(c.configurationChan, c.routinesPool)
```


## 1. 配置聚合处理

创建聚合处理实例，用于从多个配置源获取配置。

```go
// 创建配置提供器的聚合处理
func NewProviderAggregator(conf static.Providers) ProviderAggregator {
    p := ProviderAggregator{}

    if conf.File != nil {
        p.quietAddProvider(conf.File)
    }
    
    if conf.Consul != nil {
        p.quietAddProvider(conf.Consul)
    }
    // ...
}

func (p *ProviderAggregator) quietAddProvider(provider provider.Provider) {
    err := p.AddProvider(provider)
    // ...
}

// AddProvider 将多个配置提供器放到分片中
func (p *ProviderAggregator) AddProvider(provider provider.Provider) error {
    err := provider.Init()
    // ...
    if fileProvider, ok := provider.(*file.Provider); ok {
        p.fileProvider = fileProvider
    } else {
        p.providers = append(p.providers, provider)
    }
    return nil
}
```

## 2. traefix 内部配置

```go
// traefik.New(*staticConfiguration)
// 创建traefix 内部配置，给出内部默认配置
func New(staticCfg static.Configuration) *Provider {
    return &Provider{staticCfg: staticCfg}
}

// Provide allows the provider to provide configurations to traefik using the given configuration channel.
func (i *Provider) Provide(configurationChan chan<- dynamic.Message, _ *safe.Pool) error {
    // ...
    configurationChan <- dynamic.Message{
        ProviderName:  "internal",
        Configuration: i.createConfiguration(ctx),
    }

    return nil
}

// Init 初始化配置提供器，用于实现接口，不是每个配置提供器都需要初始化
func (i *Provider) Init() error {
    return nil
}

// 创建具体配置
func (i *Provider) createConfiguration(ctx context.Context) *dynamic.Configuration {
    cfg := &dynamic.Configuration{
        HTTP: &dynamic.HTTPConfiguration{
            Routers:     make(map[string]*dynamic.Router),
            Middlewares: make(map[string]*dynamic.Middleware),
            Services:    make(map[string]*dynamic.Service),
            Models:      make(map[string]*dynamic.Model),
        },
        TCP: &dynamic.TCPConfiguration{
            Routers:  make(map[string]*dynamic.TCPRouter),
            Services: make(map[string]*dynamic.TCPService),
        },
        TLS: &dynamic.TLSConfiguration{
            Stores:  make(map[string]tls.Store),
            Options: make(map[string]tls.Options),
        },
    }

    i.apiConfiguration(cfg)
    i.pingConfiguration(cfg)
    i.restConfiguration(cfg)
    i.prometheusConfiguration(cfg)
    i.entryPointModels(cfg)
    i.redirection(ctx, cfg)

    cfg.HTTP.Services["noop"] = &dynamic.Service{}

    return cfg
}
```

### 2.1 api 配置

traefix 程序管理api 注册：

```go

func (i *Provider) apiConfiguration(cfg *dynamic.Configuration) {
    if i.staticCfg.API == nil {
        return
    }

    if i.staticCfg.API.Insecure {
        cfg.HTTP.Routers["api"] = &dynamic.Router{
            EntryPoints: []string{defaultInternalEntryPointName},
            Service:     "api@internal",
            Priority:    math.MaxInt32 - 1,
            Rule:        "PathPrefix(`/api`)",
        }

        if i.staticCfg.API.Dashboard {
            cfg.HTTP.Routers["dashboard"] = &dynamic.Router{
                EntryPoints: []string{defaultInternalEntryPointName},
                Service:     "dashboard@internal",
                Priority:    math.MaxInt32 - 2,
                Rule:        "PathPrefix(`/`)",
                Middlewares: []string{"dashboard_redirect@internal", "dashboard_stripprefix@internal"},
            }
            // ...
        }

        if i.staticCfg.API.Debug {
            cfg.HTTP.Routers["debug"] = &dynamic.Router{
                EntryPoints: []string{defaultInternalEntryPointName},
                Service:     "api@internal",
                Priority:    math.MaxInt32 - 1,
                Rule:        "PathPrefix(`/debug`)",
            }
        }
    }

    cfg.HTTP.Services["api"] = &dynamic.Service{}

    if i.staticCfg.API.Dashboard {
        cfg.HTTP.Services["dashboard"] = &dynamic.Service{}
    }
}
```

### 2.2 ping 配置

traefix 程序 `ping` 接口路由配置。

```go
func (i *Provider) pingConfiguration(cfg *dynamic.Configuration) {
    if i.staticCfg.Ping == nil {
        return
    }

    if !i.staticCfg.Ping.ManualRouting {
        cfg.HTTP.Routers["ping"] = &dynamic.Router{
            EntryPoints: []string{i.staticCfg.Ping.EntryPoint},
            Service:     "ping@internal",
            Priority:    math.MaxInt32,
            Rule:        "PathPrefix(`/ping`)",
        }
    }

    cfg.HTTP.Services["ping"] = &dynamic.Service{}
}
```

## 3. 文件配置提供器

```go
// Provide 文件配置提供器，通过channel 提供配置给traefix
func (p *Provider) Provide(configurationChan chan<- dynamic.Message, pool *safe.Pool) error {
    // 从文件中加载配置
    configuration, err := p.BuildConfiguration()

    if p.Watch {
        var watchItem string

        switch {
        case len(p.Directory) > 0:
            watchItem = p.Directory
        case len(p.Filename) > 0:
            watchItem = filepath.Dir(p.Filename)
        }

        // 监听文件变动
        if err := p.addWatcher(pool, watchItem, configurationChan, p.watcherCallback); err != nil {
            return err
        }
    }

    // 将获取到的配置，发送给channel
    sendConfigToChannel(configurationChan, configuration)
    return nil
}
```

### 3.1 从文件中加载配置

```go
func (p *Provider) BuildConfiguration() (*dynamic.Configuration, error) {
    // 从目录获取配置
    if len(p.Directory) > 0 {
        return p.loadFileConfigFromDirectory(ctx, p.Directory, nil)
    }

    // 从文件获取配置
    if len(p.Filename) > 0 {
        return p.loadFileConfig(ctx, p.Filename, true)
    }
}
```

### 3.2 监听文件变动

```go
func (p *Provider) addWatcher(pool *safe.Pool, directory string, configurationChan chan<- dynamic.Message, callback func(chan<- dynamic.Message, fsnotify.Event)) error {
    watcher, err := fsnotify.NewWatcher()
    
    // ...
    err = watcher.Add(directory)
    
    // Process events
    pool.GoCtx(func(ctx context.Context) {
        defer watcher.Close()
        for {
            select {
            case evt := <-watcher.Events:
                if p.Directory == "" {
                    // ...
                    if evtFileName == confFileName {
                        callback(configurationChan, evt)
                    }
                } else {
                    callback(configurationChan, evt)
                }
        }
    })
    return nil
}
```

文件有变动，重新加载配置:

```go
func (p *Provider) watcherCallback(configurationChan chan<- dynamic.Message, event fsnotify.Event) {
    watchItem := p.Filename
    if len(p.Directory) > 0 {
        watchItem = p.Directory
    }

    // 加载配置
    configuration, err := p.BuildConfiguration()
    if err != nil {
        logger.Errorf("Error occurred during watcher callback: %s", err)
        return
    }

    sendConfigToChannel(configurationChan, configuration)
}
```

### 3.3 配置写入channel

```go
func sendConfigToChannel(configurationChan chan<- dynamic.Message, configuration *dynamic.Configuration) {
    configurationChan <- dynamic.Message{
        ProviderName:  "file",
        Configuration: configuration,
    }
}
```

## 4. Consul 配置提供器

1. consul 初始化
2. 获取consul 配置并且提供给traefix

```go
// conf.ConsulCatalog
// Init consul 初始化
func (p *Provider) Init() error {
    defaultRuleTpl, err := provider.MakeDefaultRuleTemplate(p.DefaultRule, nil)
    
    // ...
    p.defaultRuleTpl = defaultRuleTpl
    return nil
}

// Provide consul catalog 配置提供器通过channel 传递配置给traefix
func (p *Provider) Provide(configurationChan chan<- dynamic.Message, pool *safe.Pool) error {
    pool.GoCtx(func(routineCtx context.Context) {
        operation := func() error {
            // ...
            p.client, err = createClient(p.Endpoint)
            
            ticker := time.NewTicker(time.Duration(p.RefreshInterval))
            for {
                select {
                case <-ticker.C:
                    // 定时拉取最新配置
                    data, err := p.getConsulServicesData(routineCtx)

                    // ...
                    // 配置推送给traefix
                    configuration := p.buildConfiguration(routineCtx, data)
                    configurationChan <- dynamic.Message{
                        ProviderName:  "consulcatalog",
                        Configuration: configuration,
                    }
                }
            }
        }

        // 重试获取配置
        err := backoff.RetryNotify(safe.OperationWithRecover(operation), backoff.WithContext(job.NewBackOff(backoff.NewExponentialBackOff()), ctxLog), notify)
        // ...
    })

    return nil
}
```

## 参考资料

- github.com/containous/traefik/pkg/provider/aggregator/aggregator.go
- github.com/containous/traefik/pkg/provider/file/file.go
- github.com/containous/traefik/pkg/provider/consulcatalog/consul_catalog.go

