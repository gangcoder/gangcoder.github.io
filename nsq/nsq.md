<!-- ---
title: nsq 源码阅读
date: 2018-01-20 18:28:12
category: language, go, nsq
--- -->

nsq 源码阅读


* [NSQ 结构图](/nsq/nsq_structure.md)
* [NSQ 生产者实现](/nsq/go-nsq/nsq_producer.md)
* [NSQ 消费者实现](/nsq/go-nsq/nsq_consumer.md)
* [Lookupd 实现](/nsq/nsq/nsq_lookupd.md)
* [NSQD 服务实现](/nsq/nsq/nsqd.md)


## 源码阅读记录

1. 文档阅读 2018/8/11 1w
2. 源码学习 2018/8/17 2w
3. 源码笔记整理 2018/8/24 1w
4. 笔记整理 2018/9/24
5. 逻辑图整理 2020/3/23

nsq v1.2.0
go-nsq v1.0.8

- 设计
- 生产
- 消费
- 内核
- lookup 的
- 其他技术点
    - conn go-nsq/conn.go

## 文稿

内容简介
1. 有什么问题，用什么技术解决
2. 这个技术是什么样的，有哪些内容
3. 学习了能了解到什么，对你有什么帮助


安装和启动
1. 安装
2. 启动


代码情况
1. nsq 代码量和目录结构
2. nsq-go 目录结构


nsq 组成和消息模型
1. nsq 模块结构与消息投递模型
    1. 各模块功能
2. nsq 总体结构简述

源码总结构图
pruducer, consumer, nsqd, lookupd 结构图


msg 生命周期
消息投递逻辑详解


nsqd 消息处理和lookup 的数据管理详解
消息消费逻辑详解


## 图片

1. 总体架构图，nsq 结构图 (c/s, 两端，4个模块)
2. 消息消费模式图 (推送/拉取)
3. 详细结构图


## 参考资料

- [http://blog.csdn.net/qq_33649725/article/details/76977499](http://blog.csdn.net/qq_33649725/article/details/76977499)
- [初识NSQ分布式实时消息架构](http://blog.csdn.net/charn000/article/details/48109665)
- [https://www.cnblogs.com/swarmbees/p/6635467.html](https://www.cnblogs.com/swarmbees/p/6635467.html)
- [http://feixiao.github.io/2015/09/10/nsq_design/](http://feixiao.github.io/2015/09/10/nsq_design/)
- [golang使用Nsq](https://studygolang.com/articles/9749)
- [nsq.io/document](http://nsq.io/overview/design.html)
- [nsq 中文](http://wiki.jikexueyuan.com/project/nsq-guide/)
- https://github.com/nsqio/go-nsq
- https://github.com/nsqio/nsq
- http://nsq.io/overview/design.html

