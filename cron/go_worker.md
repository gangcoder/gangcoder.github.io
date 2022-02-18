<!-- ---
title: go worker
date: 2019-02-14 20:54:48
category: language, go, advance
--- -->

go worker

### 1. 实例化Func

实现 interface

FuncWorker 继承单例实现代码

- Run

Singleton 基于channel 的单例

Base 基本interface 实现

### 2. 添加配置

Options


### 3. 创建worker

NewNamedWorker 

- 初始化 cron.New
- 注册 etcd
- wrapRunner

4. 添加到App.workers 

App.workers worker.NamedWorker

Schedule 函数添加 worker

run 中启动cron

5. 运行

NamedWorker.Run

- 添加schedule
- 监听etcd 中的时间间隔配置
- 运行cron



## 参考资料

- [cron 依赖](github.com/robfig/cron)


