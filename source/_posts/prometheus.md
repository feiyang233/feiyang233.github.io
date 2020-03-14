---
title: Prometheus 指南
date: 2019-10-03 15:08:40
tags: prometheus
categories: 学习
---
Prometheus 作为业务级监控告警工具，再配合上可视化工具 Grafana，运维人员能方便的监控所需的指标。本文记录了小白入门学习的过程
<!--more-->  
# 学习资料
1. [Prometheus 中文文档](https://ryanyang.gitbook.io/prometheus/) 
2. [官方文档地址](https://prometheus.io/docs/introduction/overview/)

# 入门体验区
## 快速安装
快速安装应用首选 docker 方式，不需要配置复杂的环境。当我们已经非常熟悉如何使用 prometheus 的时候，再返回来使用普通安装。

```bash
# 只监听在本地
docker run --name prometheus -d -p 127.0.0.1:9090:9090 prom/prometheus

# 将配置文件挂载到容器中，方便修改
docker run -p 9090:9090 -v /tmp/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus

# 使用额外的数据卷挂载配置文件：
docker run -p 9090:9090 -v /prometheus-data prom/prometheus --config.file=/prometheus-data/prometheus.yml
```
安装完成后，即可访问 http://localhost:9090 可以在 Graph 查询监控项，在 Status 查看监控了哪些机器，配置文件等。

## 修改配置文件
[配置介绍官方文档]https://prometheus.io/docs/prometheus/latest/configuration/configuration/)  
我是在 Mac 电脑上 docker 安装 prometheus。由于我没有把配置文件挂载出来，只能进入容器去修改。
```bash
docker exec -u root -it 51ae3954e880 sh

vi /etc/prometheus/prometheus.yml
```
配置文件如下所示
```yaml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']

  - job_name: 'feiy'
    scrape_interval: 5s
    static_configs:
      - targets: ['host.docker.internal:8000']
        labels:
          group: 'production'
```

完整的配置例子，请点击[这里](https://github.com/prometheus/prometheus/blob/master/config/testdata/conf.good.yml)


## 检查结果
打开 http://localhost:9090/targets 我们可以看到如下的结果  

![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191003155435.png)

图中两个 target，第一个 feiy 是我自己写的一个 django 小程序，暴露出来了我关心的指标，用了[Prometheus Python Client](https://github.com/prometheus/client_python)。第二个则是 Prometheus 服务器自带的监控数据。

```bash
# feiy 暴露出来的数据

# HELP python_gc_objects_collected_total Objects collected during gc
# TYPE python_gc_objects_collected_total counter
python_gc_objects_collected_total{generation="0"} 17999.0
python_gc_objects_collected_total{generation="1"} 2384.0
python_gc_objects_collected_total{generation="2"} 833.0
# HELP python_gc_objects_uncollectable_total Uncollectable object found during GC
# TYPE python_gc_objects_uncollectable_total counter
python_gc_objects_uncollectable_total{generation="0"} 0.0
python_gc_objects_uncollectable_total{generation="1"} 0.0
python_gc_objects_uncollectable_total{generation="2"} 0.0
# HELP python_gc_collections_total Number of times this generation was collected
# TYPE python_gc_collections_total counter
python_gc_collections_total{generation="0"} 255.0
python_gc_collections_total{generation="1"} 23.0
python_gc_collections_total{generation="2"} 2.0
# HELP python_info Python platform information
# TYPE python_info gauge
python_info{implementation="CPython",major="3",minor="7",patchlevel="1",version="3.7.1"} 1.0
# HELP request_processing_seconds Time spent processing request
# TYPE request_processing_seconds summary
request_processing_seconds_count{endpoint="/metrics/",method="GET",status_code="200"} 3.0
request_processing_seconds_sum{endpoint="/metrics/",method="GET",status_code="200"} 0.0
# TYPE request_processing_seconds_created gauge
request_processing_seconds_created{endpoint="/metrics/",method="GET",status_code="200"} 1.570089497964517e+09
# HELP request_byte_sum_total Total request byte sum
# TYPE request_byte_sum_total counter
request_byte_sum_total{endpoint="/metrics/",method="GET",status_code="200"} 0.0
# TYPE request_byte_sum_created gauge
request_byte_sum_created{endpoint="/metrics/",method="GET",status_code="200"} 1.570089497964633e+09
# HELP response_byte_sum_total Total response byte sum
# TYPE response_byte_sum_total counter
response_byte_sum_total{endpoint="/metrics/",method="GET",status_code="200"} 5533.0
# TYPE response_byte_sum_created gauge
response_byte_sum_created{endpoint="/metrics/",method="GET",status_code="200"} 1.5700894979645782e+09
```

我们可以在 Graph 查询结果，比如查询 python_info{implementation="CPython",major="3",minor="7",patchlevel="1",version="3.7.1"} 1.0  
得到了如下的结果

![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20191003160311.png)

# 进阶区
## 暴露数据 node exporter
1. [监控 Linux 机器](https://prometheus.io/docs/guides/node-exporter/)
2. [监控容器](https://prometheus.io/docs/guides/cadvisor/)

## metric
1. [官方文档](https://prometheus.io/docs/concepts/metric_types/)
2. [详细解读 Prometheus 的指标类型](https://mp.weixin.qq.com/s/1EuTeQKeCX7-B2Ihj85dCA)
3. [一文搞懂 Prometheus 的直方图](https://mp.weixin.qq.com/s/WJM-DTUrdl8KFpLD5kabZg)

## rules
1. [官方文档](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/)
```bash
# 如果有错的话，则那一条规则不会更新。
go get github.com/prometheus/prometheus/cmd/promtool
promtool check rules /path/to/example.rules.yml
```
# 采坑区