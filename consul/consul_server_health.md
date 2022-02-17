<!-- ---
title: Consul Health 实现
date: 2020-06-28 23:04:35
category: showcode, consul
--- -->

# Consul Health 实现

实现Consul 健康检查逻辑。

![](images/consul_server_health.svg)

注册RPC 服务逻辑：

```go
registerEndpoint(func(s *Server) interface{} { return &Health{s} })
```

数据结构：

```go
type Health struct {
    srv *Server
}

// 定义一个健康检查
type HealthCheck struct {
    Node        string
    CheckID     types.CheckID
    Status      string
    Type        string
    // 健康检查定义
    Definition HealthCheckDefinition `bexpr:"-"`
}
```

## 1. 查询checks 列表

获取指定状态的所有checks 列表。

```go
func (h *Health) ChecksInState(args *structs.ChecksInStateRequest,
    reply *structs.IndexedHealthChecks) error {
    // 从state 中查询数据
    return h.srv.blockingQuery(
        &args.QueryOptions,
        &reply.QueryMeta,
        func(ws memdb.WatchSet, state *state.Store) error {
            
            // ...
            index, checks, err = state.ChecksInState(ws, args.State, &args.EnterpriseMeta)
            
            reply.Index, reply.HealthChecks = index, checks

            // 对结果排序
            return h.srv.sortNodesByDistanceFrom(args.Source, reply.HealthChecks)
        })
}
```

```go
// ChecksInState 从memdb 中查询数据
func (s *Store) ChecksInState(ws memdb.WatchSet, state string, entMeta *structs.EnterpriseMeta) (uint64, structs.HealthChecks, error) {
    tx := s.db.Txn(false)
    defer tx.Abort()

    // 查询数据
    idx, iter, err := s.checksInStateTxn(tx, ws, state, entMeta)
    
    // ... 
    // 遍历结果
    var results structs.HealthChecks
    for check := iter.Next(); check != nil; check = iter.Next() {
        results = append(results, check.(*structs.HealthCheck))
    }
    return idx, results, nil
}
```

## 2. 加载健康检查配置

agent 启动时，会加载健康检查配置。

```go
func (a *Agent) Start(ctx context.Context) error {
    // 加载健康检查配置
    a.loadChecks(c, nil)
}
```

```go
// loadChecks 加载健康检查配置
func (a *Agent) loadChecks(conf *config.RuntimeConfig, snap map[structs.CheckID]*structs.HealthCheck) error {
    // 加载配置文件中的健康检查配置
    for _, check := range conf.Checks {
        // 创建健康检查实例
        health := check.HealthCheck(conf.NodeName)

        // 健康检查类型
        chkType := check.CheckType()

        // ...
        // 添加健康检查
        a.addCheckLocked(health, chkType, false, check.Token, ConfigSourceLocal)
    }
    return nil
}
```

### 2.1 添加健康检查

添加一条健康检查任务。

```go
// 添加新的健康检查
func (a *Agent) addCheckLocked(check *structs.HealthCheck, chkType *structs.CheckType, persist bool, token string, source configSource) error {
    // 添加新的健康检查
    err := a.addCheck(check, chkType, service, persist, token, source)

    // ...
    return nil
}
```

### 2.2 创建健康检查任务

```go
func (a *Agent) addCheck(check *structs.HealthCheck, chkType *structs.CheckType, service *structs.NodeService, persist bool, token string, source configSource) error {
    // ...
    if chkType != nil {
        // 健康检查结果更新处理
        statusHandler := checks.NewStatusHandler(a.State, a.logger, chkType.SuccessBeforePassing, chkType.FailuresBeforeCritical)

        switch {
        // http 请求的健康检查实现
        case chkType.IsHTTP():
            // http 请求实例
            http := &checks.CheckHTTP{
                CheckID:         cid,
                ServiceID:       sid,
                HTTP:            chkType.HTTP,
                Header:          chkType.Header,
                Method:          chkType.Method,
                Body:            chkType.Body,
                Interval:        chkType.Interval,
                StatusHandler:   statusHandler,
            }

            // 健康检查处理
            http.Start()
            a.checkHTTPs[cid] = http
        // ...
        }

        // ...
    }

    return nil
}
```

节点状态更新处理。

```go
func NewStatusHandler(inner CheckNotifier, ...) *StatusHandler {
    return &StatusHandler{
        logger:                 logger,
        inner:                  inner,
    }
}
```

### 2.3 启动Http 健康检查

启动通过http 协议处理的健康检查。

```go
// Start 开启基于http 的健康检查
func (c *CheckHTTP) Start() {
    // http 请求客户端
    if c.httpClient == nil {
        // Create the HTTP client.
        c.httpClient = &http.Client{
            Timeout:   10 * time.Second,
            Transport: trans,
        }
    }

    go c.run()
}

// run 定时运行check
func (c *CheckHTTP) run() {
    // 定时器
    next := time.After(initialPauseTime)
    for {
        select {
        case <-next:
            c.check()
            next = time.After(c.Interval)
        case <-c.stopCh:
            return
        }
    }
}

// check 健康检查
func (c *CheckHTTP) check() {
    // 发送请求
    method := c.Method
    bodyReader := strings.NewReader(c.Body)
    req, err := http.NewRequest(method, target, bodyReader)

    // 请求
    resp, err := c.httpClient.Do(req)

    // 处理响应
    output, _ := circbuf.NewBuffer(int64(c.OutputMaxSize))
    io.Copy(output, resp.Body)
    
    // 根据http code 确定是否检查成功
    if resp.StatusCode >= 200 && resp.StatusCode <= 299 {
        // PASSING (2xx)
        c.StatusHandler.updateCheck(c.CheckID, api.HealthPassing, result)
    } else {
        // CRITICAL
        c.StatusHandler.updateCheck(c.CheckID, api.HealthCritical, result)
    }
}
```

### 2.4 更新节点状态

根据http 请求结果更新节点状态数据。

```go
// 更新健康请求状态
func (s *StatusHandler) updateCheck(checkID structs.CheckID, status, output string) {

    if status == api.HealthPassing || status == api.HealthWarning {
        if s.successCounter >= s.successBeforePassing {
            // ...
            s.inner.UpdateCheck(checkID, status, output)
            return
        }
    } else {
        if s.failuresCounter >= s.failuresBeforeCritical {
            // ...
            s.inner.UpdateCheck(checkID, status, output)
            return
        }
    }
}
```

更新state 中节点状态数据。

```go
// UpdateCheck 更新健康检查的状态
func (l *State) UpdateCheck(id structs.CheckID, status, output string) {
    // ...
    c := l.checks[id]

    // 更新数据
    c.Check.Status = status
    c.Check.Output = output
    c.InSync = false
    // 通知state 有更新
    l.TriggerSyncChanges()
}
```

## 3. 健康检查Http API

健康检查的http 端口处理逻辑实现。

```go
registerEndpoint("/v1/agent/check/register", []string{"PUT"}, (*HTTPServer).AgentRegisterCheck)
```

注册健康检查处理端口。

```go
func (s *HTTPServer) AgentRegisterCheck(resp http.ResponseWriter, req *http.Request) (interface{}, error) {
    // 获取健康检查参数
    health := args.HealthCheck(s.agent.config.NodeName)

    // 确定健康检查类型
    chkType := args.CheckType()
    
    // 添加健康检查
    s.agent.AddCheck(health, chkType, true, token, ConfigSourceRemote)

    return nil, nil
}

// 添加健康检查信息
func (a *Agent) AddCheck(check *structs.HealthCheck, chkType *structs.CheckType, persist bool, token string, source configSource) error {
    a.stateLock.Lock()
    defer a.stateLock.Unlock()
    return a.addCheckLocked(check, chkType, persist, token, source)
}
```

## 参考资料

- github.com/hashicorp/consul/agent/consul/health_endpoint.go
- github.com/hashicorp/consul/agent/agent.go

