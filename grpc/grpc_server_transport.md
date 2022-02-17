<!-- ---
title: Grpc 服务端传输层实现
date: 2020-10-27 23:20:02
category: showcode, grpc
--- -->

# Grpc 服务端传输层实现

主要调用逻辑：

```go
// 创建http2 协议处理
st := s.newHTTP2Transport(conn, authInfo)

// 处理请求stream
st.HandleStreams(..., ...)
```

![](images/grpc_server_transport.svg)

主要数据结构：

```go
type http2Server struct {
    conn        net.Conn
    loopy       *loopyWriter
    framer      *framer
    controlBuf *controlBuffer
    kp keepalive.ServerParameters
    activeStreams map[uint32]*Stream
}

type controlBuffer struct {
    list            *itemList
}

type Stream struct {
    id           uint32
    st           ServerTransport
    method       string
    buf          *recvBuffer
    trReader     io.Reader
    contentSubtype string
}

// 消息缓冲
type recvBuffer struct {
    c       chan recvMsg
}
```

## 1. 创建http2 连接

```go
// st := s.newHTTP2Transport(conn, authInfo)
func (s *Server) newHTTP2Transport(c net.Conn, authInfo credentials.AuthInfo) transport.ServerTransport {
    config := &transport.ServerConfig{
        KeepaliveParams:       s.opts.keepaliveParams,
    }
    
    // ...
    st, err := transport.NewServerTransport("http2", c, config)
    
    return st
}

func NewServerTransport(protocol string, conn net.Conn, config *ServerConfig) (ServerTransport, error) {
    return newHTTP2Server(conn, config)
}

func newHTTP2Server(conn net.Conn, config *ServerConfig) (_ ServerTransport, err error) {
    // ...
    
    // 创建数据帧
    framer := newFramer(conn, writeBufSize, readBufSize, maxHeaderListSize)
    // 初始化设置
    isettings := []http2.Setting{{
        ID:  http2.SettingMaxFrameSize,
        Val: http2MaxFrameLen,
    }}
    
    // http2 控制参数
    framer.fr.WriteSettings(isettings...)
    
    // http2 服务
    t := &http2Server{
        conn:              conn,
        remoteAddr:        conn.RemoteAddr(),
        localAddr:         conn.LocalAddr(),
        framer:            framer,
        activeStreams:     make(map[uint32]*Stream),
        stats:             config.StatsHandler,
        kp:                kp,
    }
    
    // 数据发送缓冲
    t.controlBuf = newControlBuffer(t.done)
    
    // 获取客户端控制帧数据
    frame, err := t.framer.fr.ReadFrame()
    sf, ok := frame.(*http2.SettingsFrame)
    // 处理客户端控制帧信息
    t.handleSettings(sf)

    go func() {
        // 处理控制帧发送缓冲
        t.loopy = newLoopyWriter(serverSide, t.framer, t.controlBuf, t.bdpEst)
        t.loopy.run()
    }()
    
    // 保活处理
    go t.keepalive()
    return t, nil
}
```

### 1.1 数据帧读写器

```go
func newFramer(conn net.Conn, writeBufferSize, readBufferSize int, maxHeaderListSize uint32) *framer {
    var r io.Reader = conn
    // ...
    w := newBufWriter(conn, writeBufferSize)
    f := &framer{
        writer: w,
        fr:     http2.NewFramer(w, r),
    }

    return f
}
```

### 1.2 处理客户端控制帧信息

```go
t.handleSettings(sf)

func (t *http2Server) handleSettings(f *http2.SettingsFrame) {
    // ...
    t.controlBuf.executeAndPut(..., &incomingSettings{
        ss: ss,
    })
}
```

### 1.3 控制帧异步处理

```go
t.loopy = newLoopyWriter(serverSide, t.framer, t.controlBuf, t.bdpEst)

func newLoopyWriter(s side, fr *framer, cbuf *controlBuffer, bdpEst *bdpEstimator) *loopyWriter {
    l := &loopyWriter{
        side:          s,
        cbuf:          cbuf,
        framer:        fr,
    }
    // ...
    return l
}
```

异步处理请求。

```go
t.loopy.run()
func (l *loopyWriter) run() (err error) {
    // ...
    for {
        // 从缓存队列获取需要处理的控制帧数据
        it, err := l.cbuf.get(true)
        
        // 处理控制帧
        l.handle(it)

        // 处理数据
        l.processData()
    }
}
```

### 1.4 保活处理

```go
go t.keepalive()

// 定时发送保活消息
func (t *http2Server) keepalive() {
    p := &ping{}
    kpTimer := time.NewTimer(t.kp.Time)
    // ...
    for {
        select {
        case <-kpTimer.C:
            // ...
            // 定时发送ping 请求保活
            t.controlBuf.put(p)
        }
    }
}
```

## 2. 控制帧缓冲

帧缓冲作用，所有数据先写入队列，再用单独goroutine 获取队列数据进行处理。

```go
t.controlBuf = newControlBuffer(t.done)

func newControlBuffer(done <-chan struct{}) *controlBuffer {
    return &controlBuffer{
        ch:   make(chan struct{}, 1),
        list: &itemList{},
        done: done,
    }
}
```

### 2.1 帧数据入队

```go
func (c *controlBuffer) put(it cbItem) error {
    _, err := c.executeAndPut(nil, it)
    return err
}

func (c *controlBuffer) executeAndPut(f func(it interface{}) bool, it cbItem) (bool, error) {
    // 数据写入队列
    c.list.enqueue(it)
    // ...
    return true, nil
}
```

### 2.2 帧数据出队与消费

```go

func (c *controlBuffer) get(block bool) (interface{}, error) {
    // ...
    h := c.list.dequeue().(cbItem)
    return h, nil
}
```

## 3. 帧数据异步处理

获取帧缓冲中的帧数据后，再次将帧数据投入队列中，并且启动函数进行异步处理。

### 3.1 处理不同帧数据

```go
// 处理控制帧
l.handle(it)

func (l *loopyWriter) handle(i interface{}) error {
    switch i := i.(type) {
    case *incomingSettings:
        return l.incomingSettingsHandler(i)
    case *dataFrame:
        return l.preprocessData(i)
    case *ping:
        return l.pingHandler(i)
    }
}
```

预处理数据帧。

```go
func (l *loopyWriter) preprocessData(df *dataFrame) error {
    str, ok := l.estdStreams[df.streamID]
    
    // ...
    str.itl.enqueue(df)
    return nil
}
```

处理ping 帧数据。

```go
func (l *loopyWriter) pingHandler(p *ping) error {
    // 直接写入 framer
    return l.framer.fr.WritePing(p.ack, p.data)

}
```

### 3.2 处理数据帧

```go
// 处理数据
l.processData()

func (l *loopyWriter) processData() (bool, error) {
    // 取出获取stream
    str := l.activeStreams.dequeue() // Remove the first stream.
    
    // 取出数据帧
    dataItem := str.itl.peek().(*dataFrame) // Peek at the first data item this stream.
    
    // ...
    buf = dataItem.d
    // 数据帧写入到传输层
    l.framer.fr.WriteData(dataItem.streamID, endStream, buf[:size])

    // ...
    return false, nil
}
```

## 4. stream 处理

```go
st.HandleStreams(..., ...)

// 处理请求流
func (t *http2Server) HandleStreams(handle func(*Stream), traceCtx func(context.Context, string) context.Context) {
    for {
        // 读取请求数据
        frame, err := t.framer.fr.ReadFrame()

        switch frame := frame.(type) {
        case *http2.MetaHeadersFrame:
            // 处理请求头帧
            t.operateHeaders(frame, handle, traceCtx)
        case *http2.DataFrame:
            // 处理数据帧
            t.handleData(frame)
        }
    }
}
```

### 4.1 处理请求头帧

```go
func (t *http2Server) operateHeaders(frame *http2.MetaHeadersFrame, handle func(*Stream), traceCtx func(context.Context, string) context.Context) (fatal bool) {
    // 解析请求头信息
    state := &decodeState{
        serverSide: true,
    }
    state.decodeHeader(frame)
    
    // 异步处理buf
    buf := newRecvBuffer()
    // 创建请求流
    s := &Stream{
        id:             streamID,
        st:             t,
        buf:            buf,
        method:         state.data.method,
        contentSubtype: state.data.contentSubtype,
    }
    
    t.activeStreams[streamID] = s

    // 请求读取
    s.trReader = &transportReader{
        reader: &recvBufferReader{
            recv:       s.buf,
        },
    }
    
    // 处理请求
    handle(s)
    return false
}
```

解析请求头信息。

```go
func (d *decodeState) decodeHeader(frame *http2.MetaHeadersFrame) (http2.ErrCode, error) {
    // ...
    for _, hf := range frame.Fields {
        d.processHeaderField(hf)
    }
}

func (d *decodeState) processHeaderField(f hpack.HeaderField) {
    switch f.Name {
    case "content-type":
        // 请求编码类型
        contentSubtype, validContentType := grpcutil.ContentSubtype(f.Value)
        d.data.contentSubtype = contentSubtype
    case ":path":
        // 请求方法信息
        d.data.method = f.Value
    }
}
```

### 4.2 异步处理buf

```go
// 创建异步读取数据buf
buf := newRecvBuffer()

func newRecvBuffer() *recvBuffer {
    b := &recvBuffer{
        c: make(chan recvMsg, 1),
    }
    return b
}
```

数据入队。

```go
func (b *recvBuffer) put(r recvMsg) {
    // ...
    b.c <- r
}
```

数据出队。

```go
func (b *recvBuffer) get() <-chan recvMsg {
    return b.c
}
```

### 4.3 处理数据帧

```go

func (t *http2Server) handleData(f *http2.DataFrame) {
    // 取到stream
    s, ok := t.getStream(f)
    
    // 数据写到异步channel
    buffer.Write(f.Data())
    s.write(recvMsg{buffer: buffer})

    // ...
}

func (s *Stream) write(m recvMsg) {
    s.buf.put(m)
}
```

### 4.4 读取请求数据

```go
// 外层读取Stream 数据
&parser{r: stream}
p.r.Read(msg)

func (s *Stream) Read(p []byte) (n int, err error) {
    // 读取数据
    return io.ReadFull(s.trReader, p)
}

func (t *transportReader) Read(p []byte) (n int, err error) {
    n, err = t.reader.Read(p)
}

func (r *recvBufferReader) Read(p []byte) (n int, err error) {
    // ...
    n, r.err = r.read(p)
    return n, r.err
}

func (r *recvBufferReader) read(p []byte) (n int, err error) {
    select {
    case m := <-r.recv.get():
        return r.readAdditional(m, p)
    }
}

func (r *recvBufferReader) readAdditional(m recvMsg, p []byte) (n int, err error) {
    // 读取数据
    copied, _ := m.buffer.Read(p)
    // ...
}
```

## 参考资料

- github.com/grpc/grpc-go/server.go

