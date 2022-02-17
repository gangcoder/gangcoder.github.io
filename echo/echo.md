<!-- ---
title: echo
date: 2017-04-08 01:19:11
category: language, go
--- -->

echo

2017/04/11 文档预览
2020/05/01 代码逻辑图整理
2020/06/17 代码阅读初步整理

## echo 使用

1. [echo 快速入门](echo/echo_quickstart.md)
    - 静态文件
    - 路由
    - context
    - cookie 2017/06/17

## echo 源码阅读

2020/05/01

echo.go //框架主体
context.go //请求上下文
bind.go //请求参数绑定
group.go //路由组定义
log.go //日志接口定义
middleware //内置中间件
response.go //http 响应封装代码
router.go //路由代码

* [Echo 代码文件结构](showcode/echo/echo_directory.md)
* [Echo 框架主逻辑](showcode/echo/echo_start.md)
* [Echo 网络监听服务](showcode/echo/echo_netlisten.md)
* [Echo 路由逻辑](showcode/echo/echo_router.md)
* [Echo 中间件](showcode/echo/echo_middleware.md)
* [Echo 请求处理](showcode/echo/echo_context.md)

## 资料

- [参考项目](https://github.com/hobo-go/echo-web)
- [echo 中文文档](http://go-echo.org/)
- [echo guide](https://echo.labstack.com/guide)
- [fast http](https://github.com/valyala/fasthttp)

