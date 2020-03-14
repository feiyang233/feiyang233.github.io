---
title: Prometheus 监控 memcached
date: 2019-04-12 10:28:32
tags:
  - docker
  - prometheus
categories: Linux
---
本文记录如何用安装 memcached_exporter 来收集 memcached 信息并且暴露给 Prometheus 监听程序，Prometheus 将收集的信息传递给 grafana 进行信息可视化。
<!--more-->
# 安装 Memcached Exporter
prometheus 官方的 memcached_exporter [文档](https://github.com/prometheus/memcached_exporter)  
## bridge 桥接方式
在 192.168.21.16 服务器上运行了三个 memcached 端口分别为 11211:11213 目前官方的这个版本还不支持多个地址，社区的解决方案[点这里](https://github.com/prometheus/memcached_exporter/issues/48)
```bash
docker run -d -p 9211:9150  --name=memcached_11211  quay.io/prometheus/memcached-exporter:v0.5.0 --memcached.address=192.168.21.16:11211

docker run -d -p 9212:9150  --name=memcached_11212  quay.io/prometheus/memcached-exporter:v0.5.0 --memcached.address=192.168.21.16:11212

docker run -d -p 9213:9150  --name=memcached_11213  quay.io/prometheus/memcached-exporter:v0.5.0 --memcached.address=192.168.21.16:11213
```
在这里我们启动了三个 docker container 用的是 bridge 网络方式来分别监听 11211--11213 需要注意的是 memcached.address 默认监听的是 localhost:11211 如果是 bridge 方式的话，用默认的方法 localhost 只能监听到容器内部。  
## host 网络方式
如果服务器只有一个 memcached 进程的话，那么我们可以用 host 网络的方式。 容器和服务器共享网络，优点是网络高性能，缺点就是需要注意端口冲突。
```
docker run --network=host  --name=memcached_11211  quay.io/prometheus/memcached-exporter:v0.5.0 --memcached.address=localhost:11211
```
##  注意 iptable
一旦使用了 docker 我们需要特别注意的就是 **iptable** 
1. ~~-A INPUT -s 172.16.0.0/12 -j DROP~~  #检查iptables filter 表 INPUT 链是否阻止了docker container IP，因为 docker 默认 IP 是 172.17.0.0/24，
2. -A INPUT -s 172.17.0.0/24 -p tcp --dport 11211:11213 -j ACCEPT  #若采用 bridge 桥接方式， 需要允许容器连接到 memcached
3. -A INPUT -s *Prometheus_IP* -p tcp --dport 9211:9213 -j ACCEPT  #给 Prometheus 开放监听的白名单  

## 检查 memcached_exporter 结果
```bash
curl 172.17.0.2:9150/metrics #直接访问容器内部
curl localhost:9211/metrics # 从docker暴露出来的端口访问

# 结果中的字段在 grafana 设置图表时，相关的图表就要用对应的字段
# 比如当前连接上 (memcached_current_connections{instance=~"$node"}) 

# TYPE memcached_connections_listener_disabled_total counter
memcached_connections_listener_disabled_total 0
# HELP memcached_connections_total Total number of connections opened since the server started running.
# TYPE memcached_connections_total counter
memcached_connections_total 255174
# HELP memcached_connections_yielded_total Total number of connections yielded running due to hitting the memcached's -R limit.
# TYPE memcached_connections_yielded_total counter
memcached_connections_yielded_total 0
# HELP memcached_current_bytes Current number of bytes used to store items.
# TYPE memcached_current_bytes gauge
memcached_current_bytes 2.57801625e+08
# HELP memcached_current_connections Current number of open connections.
# TYPE memcached_current_connections gauge
memcached_current_connections 663
# HELP memcached_current_items Current number of items stored by this instance.
# TYPE memcached_current_items gauge
memcached_current_items 1.117251e+06
# HELP memcached_items_evicted_total Total number of valid items removed from cache to free memory for new items.
# TYPE memcached_items_evicted_total counter
memcached_items_evicted_total 0
```

如果看到输出的结果，那说明 memcached_exporter 已经收集到 memcached 的信息并将此暴露出来了。
[memcached](http://www.runoob.com/memcached/memcached-stats.html) 一些字段的含义
## 常见错误
1. 配置错误 connection failed，注意地址 --memcached.address=192.168.21.16:11212 
2. 启动新的容器失败，地址端口占用，需要重启一下docker
3. iptables 一般 reload， restart 会刷新 NAT 表，导致 docker 路由失败。这种情况需要重启 docker， docker 会在 NAT 表添加路由

# Grafana 数据可视化
[grafana](https://grafana.com/docs/v3.1/datasources/prometheus/)  官方文档，添加数据源，模板。
[prometheus function](https://prometheus.io/docs/prometheus/latest/querying/functions/) 函数在画图时非常重要  
```bash
# 每分钟 command 的数量  
sum(rate(memcached_commands_total{instance=~"$node"}[1m])) by (command)
```