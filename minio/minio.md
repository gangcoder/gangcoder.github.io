---
title: minio 对象存储
date: 2018-06-12 14:06:09
category: database, storage, minio
---

minio 对象存储

## 内容学习途径

源码情况

```
-------------------------------------------------------------------------------
Language                     files          blank        comment           code
-------------------------------------------------------------------------------
Go                             484          14837          23783          94343
-------------------------------------------------------------------------------
SUM:                           661          16155          26050         104501
-------------------------------------------------------------------------------
```

学习依据（学习内容）

- [minio 源码](https://github.com/minio/minio)
- [minio go api](https://github.com/minio/minio-go)
- [minio cli](https://github.com/minio/cli)
- [https://docs.minio.io](https://docs.minio.io)
- [minio cookbook](https://github.com/minio/cookbook)


## 内容拆分

源码拆分，展示方式拆分

minio

1. minio 程序启动代码实现 2018/10/21
2. minio server 服务代码实现 2018/10/24
3. server 服务代码架构图
4. fs-v1 实现
5. cache 实现

分层结构
第一层: cli/minio server gateway
第二层: http 接口
第三层: fs-v1 / xl-v1 实现
第四层: cache 实现


## 文稿内容

1. minio_main: minio cli 代码启动和子命令
2. cli 功能与实现
3. minio_server_start: minio server 实现
4. minio server 启动代码架构图
5. minio http 服务 11/14  Configure server/xhttp.NewServer
6. minio_fs: fs-v1 实现
7. xl 实现
8. minio_cache：disk cache 实现
9. minio-go 实现
10. auth 实现


config system
NewConfigSys

policy system 11/15
NewPolicySys

notification system 
NewNotificationSys 11/16

## 参考资料

- [minio 源码](https://github.com/minio/minio)
- [minio cookbook](https://github.com/minio/cookbook)
- [minio go api](https://github.com/minio/minio-go)
- [https://docs.minio.io](https://docs.minio.io)
- [minio cli](https://github.com/minio/cli)