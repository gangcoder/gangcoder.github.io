---
title: prometheus install
date: 2018-11-17 06:52:04
category: src, prometheus
---

prometheus install

## 安装和使用

1. 安装

预编译文件

https://prometheus.io/download/

编译安装

使用Makefile ，此时调用的是 promu 进行编译

最终入口是 

服务入口

```
~gocode/src/github.com/prometheus/prometheus/cmd/prometheus/main.go
~gocode/src/github.com/prometheus/prometheus/cmd/promtool/main.go
```

web 页面入口

```
~gocode/src/github.com/prometheus/prometheus/web/web.go
```

2. 启动与监听

启动命令

```
$ ./prometheus --config.file=prometheus.yml

main.go:244 msg="Starting Prometheus" version="(version=2.5.0, branch=HEAD, revision=67dc912ac8b24f94a1fc478f352d25179c94ab9b)"
main.go:245 build_context="(go=go1.11.1, user=root@578ab108d0b9, date=20181106-11:45:24)"
main.go:246 host_details=(darwin)
main.go:247 fd_limits="(soft=7168, hard=unlimited)"
main.go:248 vm_limits="(soft=unlimited, hard=unlimited)"
main.go:562 msg="Starting TSDB ..."
web.go:399 component=web msg="Start listening for connections" address=0.0.0.0:9090
main.go:572 msg="TSDB started"
main.go:632 msg="Loading configuration file" filename=prometheus.yml
main.go:658 msg="Completed loading of configuration file" filename=prometheus.yml
main.go:531 msg="Server is ready to receive web requests."
```

3. 配置文件

prometheus.yml

https://prometheus.io/docs/prometheus/latest/configuration/configuration/

4. 指标观察

http://127.0.0.1:9090

浏览器观察

5. 配置多实例采集与持久化任务

开启多个客户端实例

```
//client_golang/examples/random

./random -listen-address=:8080
./random -listen-address=:8081
./random -listen-address=:8082
```

配置采集任务

```
scrape_configs:
  - job_name:       'example-random'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:8080', 'localhost:8081']
        labels:
          group: 'production'

      - targets: ['localhost:8082']
        labels:
          group: 'canary'
```

配置聚合查询结果为新的时间序列数据

```
//prometheus.rules.yml:

groups:
- name: example
  rules:
  - record: job_service:rpc_durations_seconds_count:avg_rate5m
    expr: avg(rate(rpc_durations_seconds_count[5m])) by (job, service)


rule_files:
  - 'prometheus.rules.yml'
```


## 参考资料

- [https://prometheus.io/docs/prometheus/latest/getting_started/](https://prometheus.io/docs/prometheus/latest/getting_started/)