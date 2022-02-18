<!-- ---
title: etcd startup
date: 2019-08-04 01:14:09
category: showcode, etcd
--- -->

# etcd 启动

## etcd 服务端

对于测试和开发部署，最快最简单的方式是搭建本地独立集群。

启动命令

```bash
$ ./etcd
```

启动的etcd 成员在 `localhost:2379` 监听客户端请求。

## etcdctl 终端工具

`etcdctl` 是一个和 etcd 服务器交互的命令行工具。

为了向后兼容，etcdctl 默认使用 v2 API 来和 etcd 服务器通讯。为了让 etcdctl 使用 v3 API 来和etcd通讯，必须通过环境变量 `ETCDCTL_API` 设置为版本3。

```bash
# 使用 API 版本 3
$ export ETCDCTL_API=3

$ ./etcdctl put foo bar
OK

$ ./etcdctl get foo
bar
```

### 写入键

应用通过写入键来储存键到etcd 中。每个存储的键通过 Raft 协议复制到 etcd 集群的所有成员来实现一致性和可靠性。

```
$ etcdctl put foo bar
OK
```

### 读取键

```
$ etcdctl get foo
foo
bar
```


只读取键 foo 的值的命令：

```
$ etcdctl get foo --print-value-only
bar
```

etcd 集群上键值存储的每个修改都会增加 etcd 集群的全局修订版本，应用可以通过提供旧有的 etcd 修改版本来读取被替代的键。

访问修订版本为 4 时的键的版本：

```
$ etcdctl get --prefix --rev=4 foo 
foo
bar_new
foo1
bar1
```

### 压缩修订版本

etcd 保存修订版本以便应用可以读取键的过往版本，为了避免积累无限数量的历史数据，etcd 通过压缩来删除历史修订版本。

```
$ etcdctl compact 5
compacted revision 5
```

在压缩修订版本之前的任何修订版本都不可访问。

### 授予租约

应用可以为 etcd 集群里面的键授予租约。当键被附加到租约时，它的存活时间被绑定到租约的存活时间，一旦租约的 TTL 到期，租约就过期并且所有附带的键都将被删除。

授予租约，TTL为10秒：

```
$ etcdctl lease grant 10
lease 32695410dcc0ca06 granted with TTL(10s)
```

附加键 foo 到租约 `32695410dcc0ca06`：

```
$ etcdctl put --lease=32695410dcc0ca06 foo bar
OK
```

## gRPC 网关

etcd grpc 网关提供 RESTful 代理，将 HTTP/JSON 请求翻译为为 gRPC 消息进行处理。

网关接受的消息字段中 `key` 和 `value` 字段被定义为 byte 数组，因此必须在 JSON 中以 base64 编码。

### 设置和获取键

使用 `v3alpha/kv/range` 和 `v3alpha/kv/put` 服务来读取和写入键:

```bash
# https://www.base64encode.org/
# foo is 'Zm9v' in Base64
# bar is 'YmFy'

curl -L http://localhost:2379/v3alpha/kv/put \
	-X POST -d '{"key": "Zm9v", "value": "YmFy"}'
# {"header":{"cluster_id":"12585971608760269493","member_id":"13847567121247652255","revision":"2","raft_term":"3"}}

curl -L http://localhost:2379/v3alpha/kv/range \
	-X POST -d '{"key": "Zm9v"}'
# {"header":{"cluster_id":"12585971608760269493","member_id":"13847567121247652255","revision":"2","raft_term":"3"},"kvs":[{"key":"Zm9v","create_revision":"2","mod_revision":"2","version":"1","value":"YmFy"}],"count":"1"}
```

## 参考资料

- []()

