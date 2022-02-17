<!-- ---
title: Consul KV 命令
date: 2019-07-23 01:06:06
category: src, consul
--- -->

# Consul KV 命令

> consul kv 子命令用于consul 键值管理。

kv 命令需要基于二级子命令使用，包括：

- `consul kv put` 设置键值对
- `consul kv get` 查询键值对
- `consul kv delete` 删除键值对
- ... 还有导出导入键值对数据

`consul kv` 子命令用于打印kv 二级子命令的帮助信息，操作consul 键值信息需要依赖 `consul kv put`, `consul kv get` 二级命令。这里我们主要讲解 `consul kv put` 命令实现逻辑。

![](images/consul_command_kv.svg)


主要代码逻辑：

```go
// 注册 consul version 子命令
Register("kv", func(cli.Ui) (cli.Command, error) { return kv.New(), nil })
Register("kv get", func(ui cli.Ui) (cli.Command, error) { return kvget.New(ui), nil })
Register("kv put", func(ui cli.Ui) (cli.Command, error) { return kvput.New(ui), nil })
```

主要代码结构：

```go
// command 可以作为一个 CLI 子命令
type Command interface {
    // Help 返回帮助信息
    Help() string

    // Run 实际执行命令的主体逻辑
    Run(args []string) int

    // Synopsis 返回使用示例信息
    Synopsis() string
}

// KV 结构包含 Client
type KV struct {
    c *Client
}
```

## 1. consul kv put 命令实现

consul kv put 用例：

```
$ consul kv put connections 5
```

与`consul version` 子命令一样，`kv put` 子命令结构实现了 `Command` 接口，在`Run` 函数中执行具体逻辑。

`Run` 函数里面，解析终端参数获取键值key 和键值数据data。再创建与agent 的通信client 实例，将键值对信息封装成 `api.KVPair` 结构后，通过client 发送给 consul agent，agent 收到键值数据后将数据同步到consul server 端。

`consul kv put` 除了直接设置键值对外，支持 `check and save` 模式，先检查数据是否存在再设置键值，如代码展示。

注意：`consul kv put` 命令是与 `consul agent` 通信。

```go
// kv put 子命令注册
Register("kv put", func(ui cli.Ui) (cli.Command, error) { return kvput.New(ui), nil })

// 创建kv 命令实例
func New(ui cli.Ui) *cmd {
    c := &cmd{UI: ui}
    c.init()
    return c
}

// kv cli 命令主体逻辑实现
func (c *cmd) Run(args []string) int {
    // 解析终端参数
    if err := c.flags.Parse(args); err != nil {
        return 1
    }

    // 获取需要put 的数据
    args = c.flags.Args()
    key, data, err := c.dataFromArgs(args)
    
    // ...
    
    // 创建http client 用来与agent 通信
    client, err := c.http.APIClient()
    
    // 格式化kv 键值对
    pair := &api.KVPair{
        Key:         key,
        ModifyIndex: c.modifyIndex,
        Flags:       c.kvflags,
        Value:       dataBytes,
        Session:     c.session,
    }

    // 终端参数控制put kv 的操作方式
    switch {
    case c.cas:
        // 先检查在保存
        ok, _, err := client.KV().CAS(pair, nil)
        // ...
        c.UI.Info(fmt.Sprintf("Success! Data written to: %s", key))
        return 0
    // ...
    default:
        // 直接保存键值对
        if _, err := client.KV().Put(pair, nil); err != nil {
            c.UI.Error(fmt.Sprintf("Error! Failed writing data: %s", err))
            return 1
        }

        c.UI.Info(fmt.Sprintf("Success! Data written to: %s", key))
        return 0
    }
}
```

## 2. 请求consul agent 实现

`client, err := c.http.APIClient()` 创建到 `consul agent` 的Client 实例。

`Client.KV()` 是对键值对操作的封装，这样键值对操作函数都在`KV` 结构体中。`KV().Put()` 用来实现键值对设置操作。

```go
// consul/api/kv.go
// KV 用来处理 k/v 请求
func (c *Client) KV() *KV {
    return &KV{c}
}

// Put 设置kv 键值对
func (k *KV) Put(p *KVPair, q *WriteOptions) (*WriteMeta, error) {
    // 设置kv 键值对
    _, wm, err := k.put(p.Key, params, p.Value, q)
    return wm, err
}

// 创建请求参数，发送请求，判断请求是否成功
func (k *KV) put(key string, params map[string]string, body []byte, q *WriteOptions) (bool, *WriteMeta, error) {
    // 创建request
    r := k.c.newRequest("PUT", "/v1/kv/"+key)
    
    // 设置请求参数
    r.setWriteOptions(q)
    for param, val := range params {
        r.params.Set(param, val)
    }

    // 发送请求
    r.body = bytes.NewReader(body)
    rtt, resp, err := requireOK(k.c.doRequest(r))
    
    // ...

    // 读取数据
    io.Copy(&buf, resp.Body)

    // 判断返回结果中是否有true
    res := strings.Contains(buf.String(), "true")

    return res, qm, nil
}
```

`put()` 函数中 `k.c.doRequest(r)` 用来发送http 请求，`requireOK()` 函数用来判断响应状态码是否是200。

## 3. kv get 实现

实现`consul kv get` 终端命令。

```go
Register("kv get", func(ui cli.Ui) (cli.Command, error) { return kvget.New(ui), nil })

func New(ui cli.Ui) *cmd {
    c := &cmd{UI: ui}
    c.init()
    return c
}
```

Run 接口实现。

```go
func (c *cmd) Run(args []string) int {
    // 解析参数
    c.flags.Parse(args)

    // ...
    // key 值
    switch len(args) {
    case 1:
        key = args[0]
    }

    // 请求consul agent 的客户端
    client, err := c.http.APIClient()
    
    switch {
    case c.keys:
        // 请求consul agent 查询键值
        keys, _, err := client.KV().Keys(key, c.separator, &api.QueryOptions{
            AllowStale: c.http.Stale(),
        })
        
        // 键值信息展示
        for _, k := range keys {
            c.UI.Info(k)
        }

        return 0
    }
}
```

请求consul agent 数据。

```go
// Keys is used to list all the keys under a prefix. Optionally,
// a separator can be used to limit the responses.
func (k *KV) Keys(prefix, separator string, q *QueryOptions) ([]string, *QueryMeta, error) {
    params := map[string]string{"keys": ""}
    
    // 获取数据
    resp, qm, err := k.getInternal(prefix, params, q)
    
    // 解析键值对
    var entries []string
    decodeBody(resp, &entries)

    return entries, qm, nil
}

func (k *KV) getInternal(key string, params map[string]string, q *QueryOptions) (*http.Response, *QueryMeta, error) {
    // http 协议请求consul agent
    r := k.c.newRequest("GET", "/v1/kv/"+strings.TrimPrefix(key, "/"))
    
    rtt, resp, err := k.c.doRequest(r)
    
    // ...
    return resp, qm, nil
}
```

## 参考资料

- github.com/hashicorp/consul/command/kv/put/kv_put.go
- github.com/hashicorp/consul/api/kv.go
- github.com/hashicorp/consul/command/kv/get/kv_get.go
