<!-- ---
title: minio xl
date: 2018-11-13 11:36:34
category: src, minio, src
--- -->

minio xl

xl: eXtensible Lattice 横向可扩展集群


封装分为三层:

1. xl-set 负责节点实例化和http api 接口封装
2. xl 实现http api 接口，调用底层存储层实现具体接口逻辑
3. posix/storage-rpc 底层存储层，负责最终存储逻辑处理

调用逻辑展示

1. 创建文件对象操作层

minio/cmd/server-main.go
newObjectLayer

waitForFormatXL 格式化xl 磁盘

newXLSets 实例化xl 集群


2. 实例化xl 集群

minio/cmd/xl-sets.go
newXLSets

xlSets

xlObjects

3. disk 加载与连接建立

connectDisks

connectEndpoint

4. 创建 StorageAPI 接口对象

minio/cmd/storage-interface.go
StorageAPI 是个接口定义可扩展底层操作接口

newPosix 本地扩展

newStorageRPC 远程扩展，基于http

5. newStorageRPC

minio/cmd/storage-rpc-client.go
newStorageRPC

minio/cmd/rpc.go
NewRPCClient

6. newPosix

minio/cmd/posix.go
newPosix


minio/cmd/xl-sets.go

7. 调用链

minio/cmd/api-router.go 注册路由

minio/cmd/xl-sets.go 路由处理逻辑
遍历xl 集群实例
调用实例封装对象 xlObjects

minio/cmd/xl-v1-bucket.go 封装对象处理
调用底层disk 处理实现: posix/storage-rpc-client 实现

8. rpc-server 服务端实现
minio/cmd/storage-rpc-server.go

封装本地 posix 操作

type storageRPCReceiver struct {
    local *posix
}



## 参考资料

