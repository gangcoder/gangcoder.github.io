<!-- ---
title: consul guide
date: 2019-07-15 14:40:02
category: src, consul, doc
--- -->

# Consul 使用指南

## 1. consul 介绍

### 1.1 功能

Consul 提供服务发现和服务配置等功能，具体包括：

- 服务发现 Consul 的客户端可用提供一个服务,比如 api 或者mysql ,另外一些客户端可用使用Consul去发现一个指定服务的提供者。
- 健康检查 Consul客户端可用提供任意数量的健康检查,指定一个服务(比如:webserver是否返回了200 OK 状态码)。
- Key/Value存储 应用程序可用根据自己的需要使用Consul的层级的Key/Value存储，比如动态配置,功能标记。
- 多数据中心: Consul支持开箱即用的多数据中心

### 1.2 基础架构

Consul是一个分布式高可用的系统。每个提供服务的实例都运行一个Consul agent。

Agent与一个或多个Consul Server 进行交互，Consul Server 用于存放和复制数据。

基础设施中需要发现其他服务的组件可以查询任何一个Consul 的server或者 agent，Agent会自动转发请求到server。


## 2. Consul 使用

### 2.1 注册服务

可以通过配置文件或者调用HTTP API来注册一个服务。

创建配置目录 (.d 后缀意思是这个路径包含了一组配置文件)。

```
mkdir /etc/consul.d
```

假设我们有一个名叫web 的服务运行在 80端口:

```
echo '{"service": {"name": "web", "tags": ["rails"], "port": 80}}' >/etc/consul.d/web.json
```

现在重启agent , 设置配置目录:

```
$ consul agent -server -bootstrap-expect 1 -data-dir /tmp/consul -node=s1 -bind=10.201.102.198 -rejoin -config-dir=/etc/consul.d/ -client 0.0.0.0
```

- data-dir：提供一个目录用来存放agent的状态，所有的agent允许都需要该目录，该目录必须是稳定的，系统重启后都继续存在


HTTP API注册服务

```
curl -X PUT -d '{"Datacenter": "dc1", "Node": "c2", "Address": "10.155.0.106", "Service": {"Service": "MAC", "tags": ["lianglian", "Mac"], "Port": 22}}' http://127.0.0.1:8500/v1/catalog/register
```

### 2.2 查询服务

DNS API:

在DNS API中，服务的DNS名字是 `NAME.service.consul`， NAME则是服务的名称。

对于上面注册的Web，它的域名是 web.service.consul

```
[root@dhcp-10-201-102-198 ~]# dig @127.0.0.1 -p 8600 web.service.consul
```

HTTP API:

HTTP API也可以用来查询服务

```
[root@dhcp-10-201-102-198 ~]# curl -s 127.0.0.1:8500/v1/catalog/service/web
```


### 2.3 健康检查

配置文件配置健康检查:

```
/etc/consul.d/web.json

{"service": {
    "name": "Faceid",
    "tags": ["extract", "verify", "compare", "idcard"],
    "address": "10.201.102.198",
    "port": 9000,
    "check": {
        "name": "ping",
        "http": "localhost:9000",
        "interval": "3s"
        }
    }
}
```

检查健康状态，可以使用HTTP API 检查在critical 状态的服务:

```
[root@dhcp-10-201-102-198 ~]# curl -s http://localhost:8500/v1/health/state/critical
```

### 2.4 K ／V 键值存储

Consul提供了一个易用的键/值存储.这可以用来保持动态配置。

PUT 请求存储Key:

```
[root@dhcp-10-201-102-198 ~]# curl -X PUT -d 'test' http://localhost:8500/v1/kv/web/key1
```

GET 请求接收key 的值:

```
[root@dhcp-10-201-102-198 ~]# curl -s http://localhost:8500/v1/kv/web/?recurse
```

## 3. Conusl 命令行

consul只有一个命令行应用，就是consul命令，consul命令可以包含agent、members 等参数进行使用。

```
[root@dhcp-10-201-102-198 ~]# consul
usage: consul [--version] [--help] <command> [<args>]
Available commands are:
    agent          Runs a Consul agent  运行一个consul agent
    event          Fire a new event
    exec           Executes a command on Consul nodes  在consul节点上执行一个命令
    info           Provides debugging information for operators  提供操作的debug级别的信息
    join           Tell Consul agent to join cluster   加入consul节点到集群中
    kv             Interact with the key-value store
    members        Lists the members of a Consul cluster    列出集群中成员
    monitor        Stream logs from a Consul agent  打印consul节点的日志信息
    watch          Watch for changes in Consul   监控consul的改变
```

- agent 维护成员的重要信息、运行检查、服务宣布、查询处理等等。
- event命令提供了一种机制，用来fire自定义的用户事件，这些事件对consul来说是不透明的，但它们可以用来构建自动部署、重启服务或者其他行动的脚本。
- exec指令提供了一种远程执行机制，比如你要在所有的机器上执行uptime命令，远程执行的工作通过job来指定，存储在KV中，agent使用event系统可以快速的知道有新的job产生。
- join指令告诉consul agent加入一个已经存在的集群中。
- members 指令输出consul agent目前所知道的所有的成员以及它们的状态，节点的状态只有alive、left、failed三种状态。
- watch 指令提供了一个机制，用来监视实际数据视图的改变(节点列表、成员服务、KV)，如果没有指定进程，当前值会被dump出来

## 4. Consul 配置

agent 的配置项可以在命令行或者配置文件进行定义。

常用命令行参数：

- bind：该地址用来在集群内部的通讯，集群内的所有节点到地址都必须是可达的，默认是0.0.0.0
- client：consul绑定在哪个client地址上，这个地址提供HTTP、DNS、RPC等服务，默认是127.0.0.1
- config-file：明确的指定要加载哪个配置文件
- data-dir：提供一个目录用来存放agent的状态，所有的agent允许都需要该目录，该目录必须是稳定的，系统重启后都继续存在
- dc：该标记控制agent 允许的datacenter 的名称，默认是dc1
- join：加入一个已经启动的agent的ip地址，可以多次指定多个agent的地，默认agent启动时不会加入任何节点。
- node：节点在集群中的名称，在一个集群中必须是唯一的，默认是该节点的主机名
- server：定义agent运行在server模式，每个集群至少有一个server，建议每个集群的server不要超过5个
- ui-dir:提供存放web ui资源的路径，该目录必须是可读的

除了命令行参数外，配置也可以写入文件。

配置文件样例：

```
{
  "datacenter": "dc1",
  "data_dir": "/opt/consul",
  "log_level": "INFO",
  "node_name": "s1",
  "server": true,
  "bootstrap_expect": 3,
  "bind_addr": "10.201.102.198",
  "client_addr": "0.0.0.0",
  "ui_dir": "/root/consul_ui",
  "retry_join": ["10.201.102.198","10.201.102.199","10.201.102.200","10.201.102.248"],
  "retry_interval": "30s",
  "enable_debug": false,
  "rejoin_after_leave": true,
  "start_join": ["10.201.102.198","10.201.102.199","10.201.102.200","10.201.102.248"],
  "enable_syslog": true,
  "syslog_facility": "local5"
}
```


## 5. HTTP API

HTTP API 可以用来增删查改nodes、services、checks、configguration。

所有的endpoints主要分为以下类别：

- kv - Key/Value存储
- agent - Agent控制
- catalog - 管理nodes和services
- health - 管理健康监测
- session - Session操作
- acl - ACL创建和管理
- event - 用户Events
- status - Consul系统状态

### 5.1 agent

agent endpoints 用来和本地agent进行交互，一般用来服务注册和检查注册。

- `/v1/agent/checks` : 返回本地agent注册的所有检查(包括配置文件和HTTP接口)
- /v1/agent/check/register : 在本地agent增加一个检查项，使用PUT方法传输一个json格式的数据
- /v1/agent/check/deregister/<checkID> : 注销一个本地agent的检查项
- `/v1/agent/services` : 返回本地agent注册的所有 服务
- `/v1/agent/members` : 返回agent在集群的gossip pool中看到的成员
- `/v1/agent/service/register` : 在本地agent增加一个新的服务项，使用PUT方法传输一个json格式的数据
- `/v1/agent/service/deregister/<serviceID>` : 注销一个本地agent的服务项

### 5.2 catalog

catalog endpoints用来注册/注销nodes、services、checks

- /v1/catalog/register : Registers a new node, service, or check
- /v1/catalog/deregister : Deregisters a node, service, or check
- /v1/catalog/datacenters : Lists known datacenters
- /v1/catalog/nodes : Lists nodes in a given DC
- /v1/catalog/services : Lists services in a given DC
- /v1/catalog/service/<service> : Lists the nodes in a given service

### 5.3 health

health endpoints用来查询健康状况相关信息。

- /v1/healt/node/<node>: 返回node所定义的检查，可用参数?dc=
- /v1/health/service/<service>: 返回给定datacenter中给定node中service
- /v1/health/service/<service>: 返回给定datacenter中给定node中service
- /v1/health/state/<state>: 返回给定datacenter中指定状态的服务，state可以是"any", "unknown", "passing", "warning", or "critical"

### 5.4 event

event endpoints用来fire新的events、查询已有的events

- /v1/event/fire/<name>: 触发一个新的event，用户event需要name和其他可选的参数，使用PUT方法
- /v1/event/list: 返回agent知道的events

### session

session endpoints用来create、update、destory、query sessions

- /v1/session/create: Creates a new session
- /v1/session/destroy/<session>: Destroys a given session
- /v1/session/info/<session>: Queries a given session
- /v1/session/node/<node>: Lists sessions belonging to a node
- /v1/session/list: Lists all the active sessions

### acl

acl endpoints用来create、update、destory、query acl

- /v1/acl/create: Creates a new token with policy
- /v1/acl/update: Update the policy of a token
- /v1/acl/destroy/<id>: Destroys a given token
- /v1/acl/info/<id>: Queries the policy of a given token
- /v1/acl/clone/<id>: Creates a new token by cloning an existing token
- /v1/acl/list: Lists all the active tokens

### status

status endpoints用来或者consul 集群的信息

- /v1/status/leader : 返回当前集群的Raft leader
- /v1/status/peers : 返回当前集群中同事


## 参考资料

- [Consul 使用手册 http://www.liangxiansen.cn/2017/04/06/consul/](http://www.liangxiansen.cn/2017/04/06/consul/)
- https://my.oschina.net/guol/blog/675281
- https://www.consul.io/docs/guides/index.html
- https://www.gitbook.com/book/vincentmi/consul-guide/details

