<!-- ---
title: Echo 请求处理
date: 2017-12-21 13:29:50
category: language, go, echo
--- -->

# Echo 请求处理

http 请求数据与响应返回基于 `Context` 实现。

![](images/echo_context.svg)

```go
// 请求Context 接口
type Context interface {
    // 请求体，包含请求头信息
    Request() *http.Request
    // 响应体，包含响应头信息
    Response() *Response
    // 响应字符串你数据
    String(code int, s string) error
    // 响应json 数据
    JSON(code int, i interface{}) error
}

// context 实例
type context struct {
    request  *http.Request // 请求体
    response *Response // 响应体
    query    url.Values // 请求查询参数
    handler  HandlerFunc // 请求对应的处理handler
}

// 响应结构
type Response struct {
    echo *Echo
    Writer http.ResponseWriter // http.ResponseWriter 的封装
    Status int //http 状态码
    Size int64 //输出数据长度
    Committed bool //输出header 信息后置为true
    ...
}
```

## 1. 请求数据解析

请求数据对应 `*http.Request`，辅助参数处理逻辑包括：

```go
// 获取url 请求参数
func (c *context) QueryParam(name string) string {
    if c.query == nil {
        c.query = c.request.URL.Query()
    }
    return c.query.Get(name)
}
```

获取POST 表单数据：

```go
// 获取请求body 里面的参数
func (c *context) FormValue(name string) string {
    return c.request.FormValue(name)
}
```

请求参数解析实现：

```go
// 将请求参数解析到传入的结构体上
func (c *context) Bind(i interface{}) error {
    return c.echo.Binder.Bind(i, c)
}

// 默认 bind 实现
func (b *DefaultBinder) Bind(i interface{}, c Context) (err error) {
    req := c.Request()

    // ...
    // 获取header 中请求参数类型
    ctype := req.Header.Get(HeaderContentType)
    switch {
    case strings.HasPrefix(ctype, MIMEApplicationJSON):
        // json
        json.NewDecoder(req.Body).Decode(i)
        // ...
    case strings.HasPrefix(ctype, MIMEApplicationXML), strings.HasPrefix(ctype, MIMETextXML):
        // xml
        xml.NewDecoder(req.Body).Decode(i)
        // ...
    case strings.HasPrefix(ctype, MIMEApplicationForm), strings.HasPrefix(ctype, MIMEMultipartForm):
        // form 表单
        params, err := c.FormParams()
        // ...
        b.bindData(i, params, "form")
    }
    return
}
```

## 2. 响应结果

`c.String` 响应http 请求内容实现：

```go
c.String(http.StatusOK, "hello world")

func (c *context) String(code int, s string) (err error) {
    // 指定响应头
    return c.Blob(code, MIMETextPlainCharsetUTF8, []byte(s))
}

// 调用 Blob 输出，设置内容类型，header 头，响应内容
func (c *context) Blob(code int, contentType string, b []byte) (err error) {
    // 设置响应header 头
    c.writeContentType(contentType)
    // 设置code
    c.response.WriteHeader(code)
    // 写入数据
    _, err = c.response.Write(b)
    return
}

func (c *context) writeContentType(value string) {
    header := c.Response().Header()
    if header.Get(HeaderContentType) == "" {
        header.Set(HeaderContentType, value)
    }
}

// 响应http 状态码
func (r *Response) WriteHeader(code int) {
    // ...
    r.Status = code
    r.Writer.WriteHeader(code)
    r.Committed = true
}

// 写入响应数据
func (r *Response) Write(b []byte) (n int, err error) {
    // ...
    n, err = r.Writer.Write(b)
    r.Size += int64(n)
    // ... 
    return
}
```

## 参考资料

- github.com/labstack/echo/context.go

