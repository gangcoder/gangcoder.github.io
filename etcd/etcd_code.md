<!-- ---
title: etcd 代码
date: 2019-08-04 01:09:26
category: showcode, etcd
--- -->

> etcd 代码与目录结构


## 代码量

代码量统计命令: `cloc ./ --exclude-dir="vendor" --include-ext="go"`

```
-------------------------------------------------------------------------------
Language                     files          blank        comment           code
-------------------------------------------------------------------------------
Go                             903          22195          24577         165534
-------------------------------------------------------------------------------
SUM:                           903          22195          24577         165534
-------------------------------------------------------------------------------
```

## 目录结构

从 `https://github.com/etcd-io/etcd` 下载etcd 代码后，需要将代码放到 `$GOPATH/src/go.etcd.io/etcd` 目录下。

```
├── Documentation // 文档目录，etcd 的文档在这里
├── auth // 权限验证处理
├── client // 服务端与客户端client 处理
├── clientv3 // 服务端与客户端client 处理v3 版本，现在建议用v3 版本
├── embed // 服务端服务使用代码
├── etcdctl // 终端工具代码，编译后是 etcdctl 工具
├── etcdmain // 服务端代码
├── etcdserver // 服务端代码，包含proto 定义和生成的pb.go 代码文件
├── lease // 租约相关代码
├── main.go
├── mvcc // etcd 多版本并发控制实现，etcd 存储的键值数据使用该工具控制存储
├── proxy // 代理工具
└── raft // etcd raft 协议实现代码
```

