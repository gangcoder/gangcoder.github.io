---
title: consul template guide
date: 2019-07-18 15:01:10
category: src, consul, doc
---

consul template guide


## 1. Consul-Template

Consul-template 可以用来动态生成配置文件。consul-template提供了一个便捷的方式从consul中获取存储的值，consul-template守护进程会查询consul实例，来更新系统上指定的任何模板。


### 1.1 使用

- auth=<user[:pass]>      设置基本的认证用户名和密码
- consul-addr=<address>   设置Consul实例的地址
- max-stale=<duration>    查询过期的最大频率，默认是1s
- dedup                   启用重复数据删除，当许多consul template实例渲染一个模板的时候可以降低consul的负载
- token=<token>           设置Consul API的token
- template=<template>     增加一个需要监控的模板，格式是：'templatePath:outputPath(:command)'，多个模板则可以设置多次
- wait=<duration>         当呈现一个新的模板到系统和触发一个命令的时候，等待的最大最小时间。如果最大值被忽略，默认是最小值的4倍。
- config=<path>           配置文件或者配置目录的路径
- dry                     Dump生成的模板到标准输出，不会生成到磁盘
- once                    运行consul-template一次后退出，不以守护进程运行


### 1.2 命令示例

查询本地consl实例，生成模板后重启nginx，如果consul不可用，如果api故障则每30s尝试检测一次值，consul-template运行一次后退出

```sh
consul-template -retry 30s -once -consul-addr=10.201.102.198:8500 -template "test.ctmpl:test.out"
```

```
// test.ctmpl
{{range service "Faceid"}}
{{.ID}} {{.Address}}:{{.Port}} check inter 5000 fall 1 rise 2 weight 2{{end}}

// test.out
Faceid 10.201.102.198:9000 check inter 5000 fall 1 rise 2 weight 2
Faceid 10.201.102.199:9000 check inter 5000 fall 1 rise 2 weight 2
Faceid 10.201.102.200:9000 check inter 5000 fall 1 rise 2 weight 2
```

### 1.3 模版语法

Consul Template 使用了Go的模板语法，可以使用Go 的语法。

## 参考资料

- [Consul 使用手册](http://www.liangxiansen.cn/2017/04/06/consul/#Consul-Template)