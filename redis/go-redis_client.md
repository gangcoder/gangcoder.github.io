<!-- ---
title: Go-redis 连接
date: 2020-11-11 13:21:03
category: showcode, redis
--- -->

# Go-redis 连接

创建 redis 连接实例，用于redis 操作。

![](images/go-redis_client.svg)

主要数据结构：

```go
// redis 连接 Options
type Options struct {
    Addr string
}

// redis 实例
type Client struct {
    *baseClient
    cmdable
    hooks
}

// 连接实例
type baseClient struct {
    opt      *Options
    connPool pool.Pooler
}

// 连接池
type ConnPool struct {
    opt *Options
    conns        []*Conn
    idleConns    []*Conn
}

// 连接
type Conn struct {
    netConn net.Conn

    rd *proto.Reader
    bw *bufio.Writer
    wr *proto.Writer
}
```

## 1. 创建redis 客户端

```go
rdb := redis.NewClient(&redis.Options{
    Addr:     "dev.com:6379",
    Password: "", // no password set
    DB:       0,  // use default DB
})
```

```go
func NewClient(opt *Options) *Client {
    opt.init()

    c := Client{
        baseClient: newBaseClient(opt, newConnPool(opt)),
        ctx:        context.Background(),
    }
    c.cmdable = c.Process

    return &c
}
```

### 1.1 配置项处理

```go
func (opt *Options) init() {
    // ...
    // 连接函数
    opt.Dialer = func(ctx context.Context, network, addr string) (net.Conn, error) {
        netDialer := &net.Dialer{
            Timeout:   opt.DialTimeout,
            KeepAlive: 5 * time.Minute,
        }
        return netDialer.DialContext(ctx, network, addr)
    }
    // ...
}
```

### 1.2 创建连接池

```go
func newConnPool(opt *Options) *pool.ConnPool {
    return pool.NewConnPool(&pool.Options{
        Dialer: func(ctx context.Context) (net.Conn, error) {
            var conn net.Conn
            err := internal.WithSpan(ctx, "dialer", func(ctx context.Context, span trace.Span) error {
                // 创建到redis 服务器的连接
                conn, err = opt.Dialer(ctx, opt.Network, opt.Addr)

                return err
            })
            return conn, err
        },
    })
}
```

### 1.3 创建客户端

```go
func newBaseClient(opt *Options, connPool pool.Pooler) *baseClient {
    return &baseClient{
        opt:      opt,
        connPool: connPool,
    }
}
```


## 2. 连接池

```go
func NewConnPool(opt *Options) *ConnPool {
    p := &ConnPool{
        opt: opt,
        conns:     make([]*Conn, 0, opt.PoolSize),
        idleConns: make([]*Conn, 0, opt.PoolSize),
    }

    // 创建连接
    p.checkMinIdleConns()
    // 回收过期连接
    go p.reaper(opt.IdleCheckFrequency)

    return p
}
```

### 2.1 创建连接

```go
func (p *ConnPool) checkMinIdleConns() {
    // ...
    for p.poolSize < p.opt.PoolSize && p.idleConnsLen < p.opt.MinIdleConns {
        go func() {
            err := p.addIdleConn()
            // ...
        }()
    }
}
```

```go
func (p *ConnPool) addIdleConn() error {
    cn, err := p.dialConn(context.TODO(), true)

    // ...
    p.conns = append(p.conns, cn)
    p.idleConns = append(p.idleConns, cn)
    return nil
}

func (p *ConnPool) dialConn(ctx context.Context, pooled bool) (*Conn, error) {
    // ...
    netConn, err := p.opt.Dialer(ctx)
    
    // ...
    cn := NewConn(netConn)
    cn.pooled = pooled
    return cn, nil
}
```

### 2.2 回收过期连接

```go
func (p *ConnPool) reaper(frequency time.Duration) {
    for {
        select {
        case <-ticker.C:
            // ...
            _, err := p.ReapStaleConns()
        }
    }
}

func (p *ConnPool) ReapStaleConns() (int, error) {
    for {
        // ...
        cn := p.reapStaleConn()
    }
    return n, nil
}

func (p *ConnPool) reapStaleConn() *Conn {
    // ...
    // 取出一条连接
    cn := p.idleConns[0]
    if !p.isStaleConn(cn) {
        return nil
    }

    p.removeConn(cn)

    return cn
}

// 连接是否过期
func (p *ConnPool) isStaleConn(cn *Conn) bool {
    // ...
    now := time.Now()
    if p.opt.IdleTimeout > 0 && now.Sub(cn.UsedAt()) >= p.opt.IdleTimeout {
        return true
    }
    if p.opt.MaxConnAge > 0 && now.Sub(cn.createdAt) >= p.opt.MaxConnAge {
        return true
    }

    return false
}

func (p *ConnPool) removeConn(cn *Conn) {
    for i, c := range p.conns {
        if c == cn {
            p.conns = append(p.conns[:i], p.conns[i+1:]...)
            return
        }
    }
}
```


## 3. 连接实例

redis 服务连接实例，完成网络数据读写。

```go
func NewConn(netConn net.Conn) *Conn {
    cn := &Conn{
        netConn:   netConn,
    }

    cn.rd = proto.NewReader(netConn)
    cn.bw = bufio.NewWriter(netConn)
    cn.wr = proto.NewWriter(cn.bw)
    return cn
}
```

### 3.1 写数据

```go
func (cn *Conn) WithWriter(
    ctx context.Context, timeout time.Duration, fn func(wr *proto.Writer) error,
) error {
    return internal.WithSpan(ctx, "with_writer", func(ctx context.Context, span trace.Span) error {
        // 
        fn(cn.wr)
        cn.bw.Flush()
        
        return nil
    })
}
```

### 3.2 读数据

```go
func (cn *Conn) WithReader(ctx context.Context, timeout time.Duration, fn func(rd *proto.Reader) error) error {
    return internal.WithSpan(ctx, "with_reader", func(ctx context.Context, span trace.Span) error {
        // ...
        fn(cn.rd)
        
        return nil
    })
}
```

## 4. 请求处理

发送redis 请求到redis 服务，并且读取redis 服务响应数据。

将Process 作为执行函数。

```go
func (c *Client) Process(ctx context.Context, cmd Cmder) error {
    return c.hooks.process(ctx, cmd, c.baseClient.process)
}
```

### 4.1 hooks 处理

hooks 处理，在处理正式命令前，执行命令处理前和处理后操作。

```go
// 执行 baseClient 命令
func (hs hooks) process(
    ctx context.Context, cmd Cmder, fn func(context.Context, Cmder) error,
) error {
    // ...
    err := hs.withContext(ctx, func() error {
        return fn(ctx, cmd)
    })

    cmd.SetErr(err)
    return err
}

func (hs hooks) withContext(ctx context.Context, fn func() error) error {
    // ...
    return fn()
}
```

### 4.2 命令执行

```go
func (c *baseClient) process(ctx context.Context, cmd Cmder) error {
    // ...
    err := c.withConn(ctx, func(ctx context.Context, cn *pool.Conn) error {
        // 发送请求
        err := cn.WithWriter(ctx, c.opt.WriteTimeout, func(wr *proto.Writer) error {
            return writeCmd(wr, cmd)
        })

        // 读取请求
        err = cn.WithReader(ctx, c.cmdTimeout(cmd), cmd.readReply)

        // ...
        return nil
    })
    // ...
    return err
}
```

### 4.3 获取连接实例

```go
func (c *baseClient) withConn(
    ctx context.Context, fn func(context.Context, *pool.Conn) error,
) error {
    return internal.WithSpan(ctx, "with_conn", func(ctx context.Context, span trace.Span) error {
        cn, err := c.getConn(ctx)

        // ...
        err = fn(ctx, cn)
        return err
    })
}

func (c *baseClient) getConn(ctx context.Context) (*pool.Conn, error) {
    // ...
    cn, err := c._getConn(ctx)
    
    // ...
    return cn, nil
}

func (c *baseClient) _getConn(ctx context.Context) (*pool.Conn, error) {
    cn, err := c.connPool.Get(ctx)
    // ...
    
    return cn, nil
}

func (p *ConnPool) Get(ctx context.Context) (*Conn, error) {
    // ...
    for {
        cn := p.popIdle()

        // ...
        return cn, nil
    }
}
```

## 参考资料

- github.com/go-redis/redis/redis.go