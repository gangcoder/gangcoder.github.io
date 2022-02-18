<!-- ---
title: nsq install
date: 2018-09-18 12:34:11
category: language, go, nsq
--- -->

nsq install

## 安装

### 下载可执行文件

可以直接下载nsq 的可执行文件使用，下载地址:

https://nsq.io/deployment/installing.html

### 源代码编译安装

```
$ git clone https://github.com/nsqio/nsq $GOPATH/src/github.com/nsqio/nsq
$ cd $GOPATH/src/github.com/nsqio/nsq
$ dep ensure
```

`dep` 是golang 的依赖管理工具


## 快速启动

以下是在本地环境，快速启动一个小型 NSQ 集群的步骤与命令

1. 启动nsqlookupd `nsqlookupd`
2. 启动nsqd `nsqd --lookupd-tcp-address=127.0.0.1:4160`
3. 启动nsqadmin `nsqadmin --lookupd-http-address=127.0.0.1:4161`
4. 发送一条消息 `curl -d 'hello world 1' 'http://127.0.0.1:4151/pub?topic=test'`
5. 消费一条消息，写入文件 `nsq_to_file --topic=test --output-dir=/tmp --lookupd-http-address=127.0.0.1:4161`
6. 从终端写入消息到nsq `./to_nsq -topic="test" -nsqd-tcp-address="127.0.0.1:4150" -rate=10`
7. 消费nsq 消息，写到终端 `./nsq_tail --topic=test --lookupd-http-address=127.0.0.1:4161`
8. 浏览器打开 `http://127.0.0.1:4171/ ` 通过nsqadmin 查看消息统计

## 参考资料

- [安装](https://nsq.io/deployment/installing.html)
- [快速启动](https://nsq.io/overview/quick_start.html)