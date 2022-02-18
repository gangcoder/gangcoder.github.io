<!-- ---
title: golang tcp server
date: 2018-08-20 21:22:13
category: language, go, nsq
--- -->

##  http server

1. 创建监听
2. 初始化处理handler
3. 开始server 监听

```go
//创建监听
n.httpListener, err = net.Listen("tcp", n.getOpts().HTTPAddress)

//新建处理函数
httpServer := newHTTPServer(ctx, false, n.getOpts().TLSRequired == TLSRequired)

//goroutine 处理http 去哪个区
n.waitGroup.Wrap(func() {
    http_api.Serve(n.httpListener, httpServer, "HTTP", n.logf)
})
```

http_api.Serve

```go
//创建server
server := &http.Server{
	Handler:  handler,
}

//进行net 监听
err := server.Serve(listener)
```

初始化处理handler

```
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}

//实现http Handler 接口
func (s *httpServer) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	s.router.ServeHTTP(w, req)
}
```


## tcp server

1. 端口监听
2. 监听处理
3. 处理逻辑

```go
//1. 端口监听
n.tcpListener, err = net.Listen("tcp", n.getOpts().TCPAddress)

//实例化处理逻辑struct
tcpServer := &tcpServer{ctx: ctx}
//2. 监听处理
go protocol.TCPServer(n.tcpListener, tcpServer, n.logf)

//定义处理逻辑接口
type TCPHandler interface {
	Handle(net.Conn)
}
//2.1 接收监听数据
func TCPServer(listener net.Listener, handler TCPHandler, logf lg.AppLogFunc) {
	for {
		clientConn, err := listener.Accept()
		go handler.Handle(clientConn)
	}
}

//3. 处理逻辑
func (p *tcpServer) Handle(clientConn net.Conn) {
	//...
}
```

## 参考资料

- [http server](#http-server)
- [tcp server](#tcp-server)
- [参考资料](#%E5%8F%82%E8%80%83%E8%B5%84%E6%96%99)