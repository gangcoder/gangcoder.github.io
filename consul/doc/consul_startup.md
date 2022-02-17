<!-- ---
title: consul 服务发现入门介绍
p: ops/consul.md
date: 2016-12-29 00:25:16
tags:
--- -->

# Consul 安装与启动

![](images/14837621814991.jpg)

Consul [ˈkɑ:nsl] 是一个分布式的服务发现和配置管理工具，本文是根据[consul 入门指南](https://www.consul.io/intro/getting-started/install.html) 整理的安装和启动笔记。

## 1. 安装

Consul 的所有命令都在一个程序包中，前往[官网下载地址 https://www.consul.io/downloads.html](https://www.consul.io/downloads.html) 下载相应系统最新的二进制包后解压就可以使用。

```
cd ~/consul
unzip consul_VERSION_linux_amd64.zip
// 解压得到 consul 二进制文件
// 将consul 软链到环境变量目录
sudo ln -s ~/consul/consul /bin/consul
// 验证是否安装成功
consul -h
```

这里在本机和1 台虚拟机中安装 consul，其中虚拟机的操作系统是CentOS 7。

### 编译安装

```sh
$ mkdir -p $GOPATH/src/github.com/hashicorp && cd $!
$ git clone https://github.com/hashicorp/consul.git
$ cd consul
$ make tools
$ make dev
//最终通过 go install 进行编译和安装
```

## 2. 运行Consul

consul 是一个CS 模式的软件，需要分别运行服务模式的consul，并且在其他节点运行客户端模式。

完成Consul的安装后,必须运行agent。agent可以运行为server或client模式。每个数据中心至少必须拥有一台server。

其他的agent运行为client模式，一个client是一个非常轻量级的进程.用于注册服务,运行健康检查和转发对server的查询。agent必须在集群中的每个主机上运行。

快速启动命令:

```sh
// 启动服务端
consul agent -server -bootstrap-expect=1 -data-dir=/tmp/consul

// 启动客户端agent 并且加入服务集群
consul agent -data-dir=/tmp/consul -join=192.168.0.104
```


### 2.1 启动server 模式

以server模式运行cosnul agent。

```
#server 模式启动
consul agent -server -bootstrap-expect=1 -data-dir=/tmp/consul -node=agent-one -bind=192.168.1.114 -config-dir=/home/dev/consul/etc
```

- `-server` 以服务模式运行
- `-node` 节点名称，默认是机器名
- `-bind` 指定监听地址，用于多网卡服务器，默认是本机IP
- `-bootstrap-expect` 额外的服务模式节点数量，启动时期望的服务模式节点数
- `config-dir` 配置文件目录
- `client` ：consul服务侦听地址，这个地址提供HTTP、DNS、RPC等服务，默认是127.0.0.1所以不对外提供服务，如果你要对外提供服务改成0.0.0.0


### 2.2 启动client 模式启动

运行cosnul agent 的client 模式，`-join` 表示要加入到已有的集群中去。

```
# client 模式启动
consul agent -data-dir=/tmp/consul -node=agent-two -bind=192.168.1.115 -config-dir=/home/dev/consul/etc
```

在集群中其他节点上以客户端模式运行consul，注意更改绑定IP 和节点名。

### 2.3 加入集群

客户端模式的consul 需要加入一个服务端节点，才能同步服务信息。

加入集群命令:

```
consul join 192.168.1.114
```

查看集群节点 `consul members` :

```
dev@ubuntu ~$ consul members
Node     Address             Status  Type    Build  Protocol  DC
agent_1  192.168.1.114:8301  alive   server  0.7.2  2         dc1
agent_2  192.168.1.115:8301  alive   client  0.7.2  2         dc1
```


通过HTTP API 查看节点信息：

```
dev@ubuntu ~$ curl localhost:8500/v1/catalog/nodes
[
    {
        "Node": "ubuntu",
        "Address": "127.0.0.1",
        "TaggedAddresses": {
            "lan": "127.0.0.1",
            "wan": "127.0.0.1"
        },
        "CreateIndex": 4,
        "ModifyIndex": 5
    }
]
```

### 2.4 服务注册与发现

服务可以通过配置文件注册，也可以通过HTTP API 添加。这里以配置文件定义服务：

```
cd ~/consul
// 创建etc 目录用于存放配置文件
mkdir etc
// 创建web.json 配置文件
echo '{"service": {"name": "web", "tags": ["nginx"], "port": 80}}' | tee ~/consul/etc/web.json
// 重启consul，并指定配置文件目录
=/home/dev/consul/etc
```

如果需要定义多个服务，添加多个服务配置文件即可，当定义服务并且重启consul 代理后，就可以通过HTTP API 查询服务信息。

```
dev@ubuntu ~/consul$ curl http://localhost:8500/v1/catalog/service/web
[
    {
        "Node": "ubuntu",
        "Address": "127.0.0.1",
        "TaggedAddresses": {
            "lan": "127.0.0.1",
            "wan": "127.0.0.1"
        },
        "ServiceID": "web",
        "ServiceName": "web",
        "ServiceTags": [
            "nginx"
        ],
        "ServiceAddress": "",
        "ServicePort": 80,
        "ServiceEnableTagOverride": false,
        "CreateIndex": 6,
        "ModifyIndex": 6
    }
]
```

## 3. 编译与开发

### 3.1 启动开发模式

为了方便开发和测试，consul 有开发者模式，可以快速开启单节点的 consul服务。

开发模式命令:

```
consul agent -dev
```

`consul members` 命令查看当前集群的节点情况

```
dev@ubuntu ~$ consul members
Node    Address         Status  Type    Build  Protocol  DC
ubuntu  127.0.0.1:8301  alive   server  0.7.2  2         dc1
```

## 参考

- [Consul 官方入门指导](https://www.consul.io/intro/getting-started/install.html)
- [Consul HTTP API](https://www.consul.io/docs/agent/http.html)
- [https://www.consul.io/docs/install/index.html](https://www.consul.io/docs/install/index.html)
- https://www.consul.io/downloads.html
