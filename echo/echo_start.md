<!-- ---
title: echo echo_process.go
date: 2017-12-21 13:29:30
category: language, go, echo
--- -->

# Echo 框架主逻辑

框架使用的主要功能逻辑：

1. 创建实例
2. 注册中间件
3. 注册路由
4. 开启服务

![](images/echo_start.svg)

主要数据结构：

```go
type Echo struct {
    middleware       []MiddlewareFunc // 中间件
    router           *Router // 路由器
    routers          map[string]*Router
    Server           *http.Server // http 处理服务
    Listener         net.Listener // http 监听处理
}

// Route 请求路由信息
type Route struct {
    Method string `json:"method"`
    Path   string `json:"path"`
    Name   string `json:"name"`
}

// http 处理服务需要实现的接口
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

## 1. 创建实例

创建 echo 实例，并且进行初始化。

```go
// github.com/labstack/echo/echo.go
// 创建echo 实例
e := echo.New()

// 创建实例函数
func New() (e *Echo) {
    e = &Echo{
        Server:    new(http.Server),
    }
    // ...
    e.Server.Handler = e
    e.Binder = &DefaultBinder{}
    e.router = NewRouter(e)
    e.routers = map[string]*Router{}
    return
}
```

`*Echo` 实例作为http 服务的 `Handler` 需要实现 `ServeHTTP(ResponseWriter, *Request)` 方法，用于处理http 请求。

1. 根据请求参数和路由树，获取处理句柄，倒序取出中间件，对处理函数进行包装
2. 执行中间件和处理函数

```go
// ServeHTTP 实现http 接口，用于处理http 请求
func (e *Echo) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 查找路由
    e.findRouter(r.Host).Find(r.Method, GetPath(r), c)
    // 获取处理函数
    h = c.Handler()
    // 从最后一个中间件往前，开始获取中间件函数
    h = applyMiddleware(h, e.middleware...)

    // 最后执行所有中间件和逻辑处理handler 
    if err := h(c); err != nil {
        e.HTTPErrorHandler(err, c)
    }
}
```

```go
// 中间件调用
// 调用中间件逻辑，返回中间件处理后的handler
func applyMiddleware(h HandlerFunc, middleware ...MiddlewareFunc) HandlerFunc {
    for i := len(middleware) - 1; i >= 0; i-- {
        // 从后往前，取出中间件逻辑执行，产生handler
        h = middleware[i](h)
    }
    return h
}
```

## 2. 注册中间件

```go
// 注册中间件
e.Use(middleware.Recover())

// 添加中间件
func (e *Echo) Use(middleware ...MiddlewareFunc) {
    e.middleware = append(e.middleware, middleware...)
}
```


## 3. 注册路由

注册路由，路由参数包括：路由路径，处理函数和中间件。

```go
// 注册Get 路由
e.GET("/", hello)

func (e *Echo) GET(path string, h HandlerFunc, m ...MiddlewareFunc) *Route {
    return e.Add(http.MethodGet, path, h, m...)
}
```

调用框架路由实例的Add 方法，基于请求路径，将handler 放到路由树上。

```go
// Add 将路由参数注册到路由器上
func (e *Echo) Add(method, path string, handler HandlerFunc, middleware ...MiddlewareFunc) *Route {
    return e.add("", method, path, handler, middleware...)
}

func (e *Echo) add(host, method, path string, handler HandlerFunc, middleware ...MiddlewareFunc) *Route {
    // 获取handler 名称
    name := handlerName(handler)
    // 查找路由器实例
    router := e.findRouter(host)
    // 将路由参数添加到路由器上
    router.Add(method, path, func(c Context) error {
        h := handler
        // 注意执行中间件
        for i := len(middleware) - 1; i >= 0; i-- {
            h = middleware[i](h)
        }
        return h(c)
    })
    // 添加路由信息
    r := &Route{
        Method: method,
        Path:   path,
        Name:   name,
    }
    e.router.routes[method+path] = r
    return r
}
```

## 4. 开启服务

设置监听地址，开启服务 `e.Start()`

```go
// 开启监听服务
e.Start(":1323")

func (e *Echo) Start(address string) error {
    e.Server.Addr = address
    return e.StartServer(e.Server)
}

// 开启网络监听
// 调用http.Server 处理网络监听
func (e *Echo) StartServer(s *http.Server) (err error) {
    // 处理handler
    s.Handler = e
    
    // 开启网络监听
    e.Listener, err = newListener(s.Addr)

    // 开启http 服务
    return s.Serve(e.Listener)
}

// 创建网络监听
func newListener(address string) (*tcpKeepAliveListener, error) {
    l, err := net.Listen("tcp", address)
    // 类型断言
    return &tcpKeepAliveListener{l.(*net.TCPListener)}, nil
}
```

## 参考资料

- github.com/labstack/echo/echo.go

