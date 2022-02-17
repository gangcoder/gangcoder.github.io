<!-- ---
title: traefix 安装与启动
date: 2020-07-11 12:18:38
category: showcode, gateway, traefix
--- -->

# traefix 安装与启动

## 安装Traefix

下载地址: https://github.com/containous/traefik/releases

## 配置示例

Traefik中的配置可以引用两个不同的内容：

1. 完全动态路由配置（称为动态配置 ）
2. 启动配置（称为静态配置 ）

静态配置中的元素建立到提供程序的连接，并定义Traefik将侦听的入口点（这些元素不会经常更改）.

动态配置包含定义系统如何​​处理请求的所有内容. 此配置可以更改，并且可以无缝地热重载，而不会导致任何请求中断或连接丢失.

Traefik从提供者那里获取其动态配置 ：编排器，服务注册表还是普通的旧配置文件。


## 服务发现

Traefik中的配置发现是通过提供商实现的.

提供程序是现有的基础结构组件，无论是协调器，容器引擎，云提供程序还是键值存储. 这个想法是Traefik将查询提供商的API，以查找有关路由的相关信息，并且Traefik每次检测到更改时，都会动态更新路由.

## 文件提供者

文件提供程序使您可以在TOML或YAML文件中定义动态配置。

```toml
[providers]
  [providers.file]
    filename = "/path/to/config/dynamic_conf.toml"
```


静态配置：

```toml
[entryPoints]
  [entryPoints.web]
    # Listen on port 8081 for incoming requests
    address = ":8081"

[providers]
  # Enable the file provider to define routers / middlewares / services in file
  [providers.file]
    directory = "/path/to/dynamic/conf"
```

动态配置：

```
# http routing section
[http]
  [http.routers]
     # Define a connection between requests and services
     [http.routers.to-whoami]
      rule = "Host(`example.com`) && PathPrefix(`/whoami/`)"
      # If the rule matches, applies the middleware
      middlewares = ["test-user"]
      # If the rule matches, forward to the whoami service (declared below)
      service = "whoami"

  [http.middlewares]
    # Define an authentication mechanism
    [http.middlewares.test-user.basicAuth]
      users = ["test:$apr1$H6uskkkW$IgXLP6ewTrSuBkTrqE8wj/"]

  [http.services]
    # Define how to reach an existing service on our infrastructure
    [http.services.whoami.loadBalancer]
      [[http.services.whoami.loadBalancer.servers]]
        url = "http://private/whoami-service"
```


## 控制台

仪表板与API位于同一位置，但默认情况下位于/dashboard/路径上.

```
[api]
  dashboard = true
  insecure = true
```


## 日志

默认情况下，日志以文本格式写入标准输出.

```
# Configuring a buffer of 100 lines
[accessLog]
  filePath = "/path/to/access.log"
  bufferingSize = 100
```


## 参考资料

- https://docs.traefik.io/getting-started/install-traefik/#compile-your-binary-from-the-sources

