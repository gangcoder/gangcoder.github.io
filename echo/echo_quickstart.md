---
title: echo 一: 快速入门
date: 2017-04-08 01:12:30
category: language, go, web
---

echo 一: 快速入门

## 安装

```
$ cd <project in $GOPATH>
$ go get github.com/labstack/echo/...

# 安装 autocert 依赖
$ go get github.com/golang/crypto
$ mv github.com/golang/crypto/ golang.org/x/
```

## 静态文件

`Static(prefix, root string)` 用一个 url 路径注册一个新的路由来提供静态文件的访问服务

```
e := echo.New()
e.Static("/static", "assets")
```

将所有访问 `/static/*` 的请求去访问 `assets` 目录。

`File(path, file string)` 使用 url 路径注册一个新的路由去访问某个静态文件

```
e.File("/", "public/index.html")
```

将 `public/index.html` 作为主页

## 路由

路由通过特定的 HTTP 方法，url 路径和一个匹配的 handler 来注册。

```
// 业务处理
func hello(c echo.Context) error {
  	return c.String(http.StatusOK, "Hello, World!")
}

// 路由
e.GET("/hello", hello)

// 访问方式为 Get，访问路径为 /hello，处理结果是返回输出 Hello World 的响应。
```

`Echo.Any(path string, h Handler)` 来为所有的 HTTP 方法注册 handler

`Echo.Match(methods []string, path string, h Handler)` 为某些方法注册路由

### 路由参数匹配

- `/users/new` 匹配完整路径
- `/users/:id` 匹配参数
- `/users/*` 将会匹配所有

优先级从上到下

### 组路由

```
Echo#Group(prefix string, m ...Middleware) *Group
```

拥有相同前缀的路由可以通过中间件定义一个子路由来化为一组。

除了一些特殊的中间件，组路由也会继承父中间件。

在组路由里使用中间件可以用 `Group.Use(m ...Middleware)` 注册。

```
g := e.Group("/admin")
g.Use(middleware.BasicAuth(func(username, password string) bool {
	if username == "joe" && password == "secret" {
		return true
	}
	return false
}))
// 创建了一个 admin 组，使所有的 /admin/* 都要求 HTTP 基本认证
```

## context

扩展context 分三步:

1. 自定义context

```
type CustomContext struct {
    echo.Context
}

func (c *CustomContext) Foo() {
    println("foo")
}

func (c *CustomContext) Bar() {
    println("bar")
}
```

2. 创建中间件扩展context

```
e.Use(func(h echo.HandlerFunc) echo.HandlerFunc {
    return func(c echo.Context) error {
        cc := &CustomContext{c}
        return h(cc)
    }
})
```

3. 使用

```
e.Get("/", func(c echo.Context) error {
	cc := c.(*CustomContext)
	cc.Foo()
	cc.Bar()
	return cc.String(200, "OK")
})
```

## cookie

设置cookie

```
func wcookie(c echo.Context) error {
	cookie := new (http.Cookie)
	cookie.Name = "username"
	cookie.Value = "jon"
	cookie.Expires = time.Now().Add(24*time.Hour)
	c.SetCookie(cookie)

	return c.String(http.StatusOK, "write a cookie")
}
```

读取cookie

```
func rcookie (c echo.Context) error {
	cookie, err := c.Cookie("username")
	if err != nil {
		return err
	}

	fmt.Println(cookie.Name)
	fmt.Println(cookie.Value)

	return c.String(http.StatusOK, "read a cookie")
}
```

## 参考

- [echo guide](https://echo.labstack.com/guide)

