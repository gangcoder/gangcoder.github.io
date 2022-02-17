---
title: go redis src conn
date: 2019-01-22 21:23:28
category: src, redis
---

go redis src conn

- 对象结构
- 连接处理
- 连接配置
- 发送请求
- pipeline 命令
- net 读取与写入
  - net 读取数据
  - 命令写入 net


## 对象结构

conn 连接结构

```go
// conn is the low-level implementation of Conn
type conn struct {
	pending int //pipeline 命令数，用于对齐pipeline 请求返回数
	conn    net.Conn //网络连接

	// Read
	readTimeout time.Duration
	br          *bufio.Reader

	// Write
	writeTimeout time.Duration
	bw           *bufio.Writer
}
```

dial 连接配置

```go
type dialOptions struct {
	readTimeout  time.Duration
	writeTimeout time.Duration
	dialer       *net.Dialer //连接器
	dial         func(network, addr string) (net.Conn, error) //连接函数
	db           int
	password     string
}
```


## 连接处理

基于 network 网络协议，连接到redis 服务层

```go
func Dial(network, address string, options ...DialOption) (Conn, error) {
	//处理dial 配置
    do := dialOptions{
		dialer: &net.Dialer{
			KeepAlive: time.Minute * 5,
		},
	}
    
    //dial 函数处理
	if do.dial == nil {
		do.dial = do.dialer.Dial
	}

    //建立网络连接
	netConn, err := do.dial(network, address)
	
    //如果启用tls 还需要升级协议
	if do.useTLS {
		//...
		netConn = tlsConn
	}

    //创建连接实例
	c := &conn{
		conn:         netConn,
		bw:           bufio.NewWriter(netConn),
		br:           bufio.NewReader(netConn),
		readTimeout:  do.readTimeout,
		writeTimeout: do.writeTimeout,
	}

    //处理password 
	if do.password != "" {
		if _, err := c.Do("AUTH", do.password); err != nil {
			netConn.Close()
			return nil, err
		}
	}

	return c, nil
}
```


## 连接配置

通过函数设置连接配置信息，这种方式方便处理配置合并的逻辑

```go
// 配置结构，里面是个函数
type DialOption struct {
	f func(*dialOptions)
}

// 配置函数示例，返回配置结构体
func DialReadTimeout(d time.Duration) DialOption {
	return DialOption{func(do *dialOptions) {
		do.readTimeout = d
	}}
}

//调用示例
func DialTimeout(network, address string, connectTimeout, readTimeout, writeTimeout time.Duration) (Conn, error) {
	return Dial(network, address,
		DialConnectTimeout(connectTimeout),
		DialReadTimeout(readTimeout),
		DialWriteTimeout(writeTimeout))
}

// 配置解析
func Dial(network, address string, options ...DialOption) (Conn, error) {
	do := dialOptions{
		dialer: &net.Dialer{
			KeepAlive: time.Minute * 5,
		},
	}
	for _, option := range options {
		option.f(&do)
	}
    
    //...
}
```


## 发送请求

Do 发起请求，并且获取服务端响应

```go
func (c *conn) Do(cmd string, args ...interface{}) (interface{}, error) {
	return c.DoWithTimeout(c.readTimeout, cmd, args...)
}

//处理超时设置
func (c *conn) DoWithTimeout(readTimeout time.Duration, cmd string, args ...interface{}) (interface{}, error) {
    //设置命令写入超时
	if c.writeTimeout != 0 {
		c.conn.SetWriteDeadline(time.Now().Add(c.writeTimeout))
	}

    //写入命令
	if cmd != "" {
		if err := c.writeCommand(cmd, args); err != nil {
			return nil, c.fatal(err)
		}
	}

    //将命令刷入服务端
	if err := c.bw.Flush(); err != nil {
		return nil, c.fatal(err)
	}

    //设置读取响应超时时间
	c.conn.SetReadDeadline(deadline)

    //正常pipeline 请求，返回服务端最后一次响应的数据
	var err error
	var reply interface{}
	for i := 0; i <= pending; i++ {
		var e error
		if reply, e = c.readReply(); e != nil {
			return nil, c.fatal(e)
		}
		if e, ok := reply.(Error); ok && err == nil {
			err = e
		}
	}
	return reply, err
}
```


## pipeline 命令

- Send 发送pipeline 命令
- Flush 将所有命令写入服务端
- Receive 读取一条响应
- Do 读取服务端返回的所有响应数据

```go
//Send 发送pipeline 命令
func (c *conn) Send(cmd string, args ...interface{}) error {
    c.pending += 1 //发送命令数计数
    
	//设置超时
	c.conn.SetWriteDeadline(time.Now().Add(c.writeTimeout))
    
    //写入命令
	if err := c.writeCommand(cmd, args); err != nil {
		return c.fatal(err)
	}
	return nil
}

//Flush 将所有命令写入服务端
func (c *conn) Flush() error {
	//将所有命令写入服务端
	if err := c.bw.Flush(); err != nil {
		return c.fatal(err)
	}
	return nil
}

//Receive 读取一条响应
func (c *conn) Receive() (interface{}, error) {
	return c.ReceiveWithTimeout(c.readTimeout)
}

func (c *conn) ReceiveWithTimeout(timeout time.Duration) (reply interface{}, err error) {
	//设置读超时
	c.conn.SetReadDeadline(deadline)

    //读取响应数据
	if reply, err = c.readReply(); err != nil {
		return nil, c.fatal(err)
    }
    
    //读取响应后，计数器 -1
	c.mu.Lock()
	if c.pending > 0 {
		c.pending -= 1
	}
	c.mu.Unlock()
	if err, ok := reply.(Error); ok {
		return nil, err
	}
	return
}

//Do 读取服务端返回的所有响应数据
func (c *conn) Do(cmd string, args ...interface{}) (interface{}, error) {
	return c.DoWithTimeout(c.readTimeout, cmd, args...)
}

//处理超时设置
func (c *conn) DoWithTimeout(readTimeout time.Duration, cmd string, args ...interface{}) (interface{}, error) {
    //为了处理pipeline 最后调用Do 收尾的情况
    c.mu.Lock()
	pending := c.pending //对应发送了多少pipeline 命令
	c.pending = 0
	c.mu.Unlock()

    //设置读取响应超时时间
	c.conn.SetReadDeadline(deadline)

    //如果没有cmd 说明是pipeline 收尾处理，读取所有数据返回
	if cmd == "" {
		reply := make([]interface{}, pending)
		for i := range reply {
			r, e := c.readReply()
			if e != nil {
				return nil, c.fatal(e)
			}
			reply[i] = r
		}
		return reply, nil
	}
}
```

## net 读取与写入

### net 读取数据

- readLine 从net 连接读取数据
- readReply 解析redis 服务端响应的数据


```go
//readLine 从net 连接读取数据
func (c *conn) readLine() ([]byte, error) {
	// 读取数据
	p, err := c.br.ReadSlice('\n')
	if err == bufio.ErrBufferFull {
		// 如果buf 可以读满，说明数据没有读取结束，此时循环读取直到读完
		buf := append([]byte{}, p...)
		for err == bufio.ErrBufferFull {
			p, err = c.br.ReadSlice('\n')
			buf = append(buf, p...)
		}
		p = buf
	}
	if err != nil {
		return nil, err
    }
    
    //判断读取的数据格式是否正确
	i := len(p) - 2
	if i < 0 || p[i] != '\r' {
		return nil, protocolError("bad response line terminator")
	}
	return p[:i], nil
}

//readReply 解析redis 服务端响应的数据
func (c *conn) readReply() (interface{}, error) {
	//读取内容
    line, err := c.readLine()
    
    //解析内容
	switch line[0] {
	case '+':
		//...
	case '*':
		//...
		return r, nil
	}
	return nil, protocolError("unexpected response line")
}
```

### 命令写入 net

```go
//writeCommand 将命令写入redis server
func (c *conn) writeCommand(cmd string, args []interface{}) error {
	c.writeLen('*', 1+len(args))
	if err := c.writeString(cmd); err != nil {
		return err
	}
	for _, arg := range args {
		if err := c.writeArg(arg, true); err != nil {
			return err
		}
	}
	return nil
}

//处理命令参数写入 redis server
func (c *conn) writeArg(arg interface{}, argumentTypeOK bool) (err error) {
	switch arg := arg.(type) {
	case string:
		return c.writeString(arg)
	case []byte:
		//...
	default:
		//...
		var buf bytes.Buffer
		fmt.Fprint(&buf, arg)
		return c.writeBytes(buf.Bytes())
	}
}
```


## 参考资料

- [redigo\redis\conn.go](redigo\redis\conn.go)