---
title: 计算机网络相关
date: 2019-05-30 11:21:34
tags: network
categories: Linux
---
记录网络相关知识与 Linux 网络相关的命令。
<!--more-->
# reference
1. [TCP/IP 三次握手与四次挥手](https://mp.weixin.qq.com/s/TxcX8Xlp62Rpqo9NEsoLqA)
2. [HTTPS 原理分析](https://mp.weixin.qq.com/s/yuwZ24gMMZsLta3psgYuPg)
3. [Iptables其底层架构 Netfilter](https://mp.weixin.qq.com/s/XmCMkgT_F5-bfHzxYQYsJQ)
4. [面对 DDoS，实现在 1 秒内丢弃掉 1000 万个网络数据包攻击](https://mp.weixin.qq.com/s/TLeKpbHMLZ4hBFcz3vjbsg)
5. [3 个 Linux 中快速检测端口的小技巧](https://mp.weixin.qq.com/s/HDHK_U6aTspzXMUgNyQ91g)

# Linux 命令
1. [route](http://www.cnblogs.com/peida/archive/2013/03/05/2943698.html) 某些 IP 段故障
2. [mtr](https://blog.einverne.info/post/2017/11/mtr-usage.html) 连通性测试
3. [Linux 常用网络指令](http://cn.linux.vbird.org/linux_server/0140networkcommand_1.php)
4. [iptables 奥哥篇](https://wsgzao.github.io/post/iptables/)
5. [iptables-NAT](http://cn.linux.vbird.org/linux_server/0250simple_firewall_5.php)
6. [iperf](http://man.linuxde.net/iperf) [测试网速](https://docs.azure.cn/zh-cn/articles/azure-operations-guide/virtual-network/aog-virtual-network-iperf-bandwidth-test)的工具
7. 删除网卡 ip link delete  [网卡名称]
8. 检查未被正确关闭的文件 [lsof](https://www.cnblogs.com/276815076/p/3503923.html) 或者 [linux中如何解决文件已删除但空间不释放的案例](https://mp.weixin.qq.com/s/o3Q-aiOWbn8s4jrwpf-4Qg)
9. 磁盘监控命令[iostat](https://www.cnblogs.com/peida/archive/2012/12/28/2837345.html)
10. 高 disk IO 检查: **iotop -oP**  看哪个进程占用大量的 disk IO
11. 查看链接状态
```bash
netstat -n | awk '/^tcp/ {++state[$NF]} END {for(key in state) print key,"\t",state[key]}'
# 如下结果
FIN_WAIT2 	 5683
SYN_RECV 	 311
CLOSE_WAIT 	 3
TIME_WAIT 	 127952
ESTABLISHED 	 52032
FIN_WAIT1 	 22345

```
12. What is a TCP Reset [RST](https://www.corvil.com/kb/what-is-a-tcp-reset-rst)  


## iptables    
```
iptables -t  nat  -nL   # 查看 nat 表
iptables -t 表名 -N 自定义链名 # 创建一个链
iptables -t 表名 -L  default filter
iptables -t 表名 -L 链名
iptables -t 表名 -nL --line
iptables -t 表名 -D 链名 要删除的序号
iptables -t 表名 -P 链名 动作    修改默认规则 DROP
-A INPUT -m iprange --src-range x.x.x.x-x.x.x.x  -p tcp --dport 11211 -j ACCEPT #多 IP
-A INPUT -p tcp -m multiport --dports 80,443 -j ACCEPT  #多端口
类型匹配 -p tcp udp udplite icmp icmpv6 esp ah sctp mh
转发功能 cat /proc/sys/net/ipv4/ip_forward
```
# 小知识
## Public IP
- Class A: 0.x.x.x ~ 127.x.x.x
- Class B: 128.x.x.x ~ 191.x.x.x
- Class C: 192.x.x.x ~ 223.x.x.x
- Class D: 224.x.x.x ~ 239.x.x.x  #multicast
- Class E: 240.x.x.x ~ 255.x.x.x  #保留

## Private IP 
+ Class A: 10.0.0.0 ~ 10.255.255.255
+ Class B: 172.16.0.0 ~ 172.31.255.255
+ Class C: 192.168.0.0 ~ 192.168.255.255 

+ 169.254.x.x  临时 IP DHCP is full. 就用这个 IP 
## Loopback IP
+ Class A: 127.0.0.1/8


# 设置 NAT server
开启转发功能
```bash
vim /etc/sysctl.conf

net.ipv4.ip_forward=1 # 添加此行，开启转发功能

sysctl -p    # 执行生效
```
还需要在 iptable 里设置转发规则
```bash
vim /etc/sysconfig/iptables

*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -o em2 -j MASQUERADE
COMMIT

# em2 是公网网卡，当其他内网机器设置 NAT 机器的内网 IP 为网关时。 
# 内网机器发包给 NAT 机器， NAT 机器根据路由规则，将会由 em2 公网网卡转发出去。
# 转发时，会将包的源 IP 替换为自己公网的 IP
```

# 网络扫描
安装 [nmap 官网](https://nmap.org/download.html) 
```bash
rpm -vhU https://nmap.org/dist/nmap-7.80-1.x86_64.rpm
# yum install nmap 版本比较老
#主机发现 ICMP ARP 两种方式
nmap -v -n -sn -PE 192.168.21.0/24
# -v 指定详细输出
# -n 不进行 DNS 解析
# -sn 使用 ping 扫描，禁用端口扫描
# -PE 指定使用 ICMP Echo Request 发现主机
# 192.168.21.0/24 为目标网段
# -PR 指定使用 ARP Request 发现主机

# TCP Connect 端口扫描
nmap  -v -n -sT --max-retries 1 -p1-65535 192.168.21.19

# -sT 使用 TCP Connect 
# --max-retries 每个端口最多重试的次数
# -p1-65535 指定端口的扫描范围
# 192.168.21.19 为被扫描主机的 IP

# TCP SYN 端口扫描,导致服务器出现大量的半连接
nmap  -v -n -sS --max-retries 1 -p1-65535 192.168.21.19

# -sS 使用 TCP SYN 

# UDP 扫描
nmap  -v -n -sU --max-retries 1 -p1-65535 192.168.21.19
# -sU 使用 UDP 进行扫描

# 识别指定端口应用
nmap  -v -n -sV -p2333 192.168.21.19
# -sV 指定识别该端口上的应用
```

# tcpdump 抓包分析
+ 进入 INPUT 的流量不会被 iptable 影响
+ 出口 OUTPIT 流量会受到 iptable 影响

# ICMP
+ ping 默认发 4、5 个包
+ traceroute 显示的是不同 AS 之间的跳数。其实一个 AS 内部可能有很多路由器，TTL 与实际跳数是不符合的。

# TCP 
+ 三次握手建立连接
+ 四次分手，等待 2 MSL
+ 只走一条路径
+ [详细介绍](https://mp.weixin.qq.com/s/TxcX8Xlp62Rpqo9NEsoLqA)

# UDP
+ 更轻更快
+ 多条路径同时发送

# DDos 攻击
DDOS 攻击，它在短时间内发起大量请求，耗尽服务器的资源，无法响应正常的访问，造成网站实质下线。DDOS 里面的 DOS 是 denial of service（停止服务）的缩写，表示这种攻击的目的，就是使得服务中断。最前面的那个 D 是 distributed （分布式），表示攻击不是来自一个地方，而是来自四面八方，因此更难防。
1. 比较常见的一种攻击是 cc 攻击。它就是简单粗暴地送来大量正常的请求，超出服务器的最大承受量，导致宕机。
2. SYN攻击属于DoS攻击的一种，它利用TCP协议缺陷，通过发送大量的半连接请求，耗费CPU和内存资源。SYN攻击除了能影响主机外，还可以危害路由器、防火墙等网络系统，事实上SYN攻击并不管目标是什么系统，只要这些系统打开TCP服务就可以实施。服务器接收到连接请求（syn= j），将此信息加入未连接队列，并发送请求包给客户（syn=k,ack=j+1），此时进入SYN_RECV状态。当服务器未收到客户端的确认包时，重发请求包，一直到超时，才将此条目从未连接队列删除。配合IP欺骗，SYN攻击能达到很好的效果，通常，客户端在短时间内伪造大量不存在的IP地址，向服务器不断地发送syn包，服务器回复确认包，并等待客户的确认，由于源地址是不存在的，服务器需要不断的重发直至超时，这些伪造的SYN包将长时间占用未连接队列，正常的SYN请求被丢弃，目标系统运行缓慢，严重者引起网络堵塞甚至系统瘫痪。


# 踩过的坑
1. docker container IP default  is 172.17.0.0/16  检查 iptables 是否阻挡
2. -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT 这一条 iptables 的规则也非常重要。Packets in a RELATED or ESTABLISHED state are those ones which belong to an already opened connection; you'll generally want to accept them, otherwise connections will get established correctly but nothing will be able to flow after the initial handshake. 如果没有这一条，会遇到 DNS 解析失败， curl 失败。 凡是 iptables 没有允许的 IP, 都不能正常的工作。 例如 DNS 查询发包后，三次握手建立。回包收到了，却会被 iptables 阻挡，上层应用无法拿到解析的结果，导致 hang 住。
3. 抓包分析时， 进入的包都可以抓到，不会受到 iptables, 发出的包会受到 iptables 影响，可能被 iptables 阻挡导致抓包失败。
4. 在此衷心的感谢，皇族后裔，八旗子弟，爱新觉罗·高Li，提供的帮助！
5. IP 冲突，导致服务不稳定。需要使用 arping -c 3  -I em2 192.168.1.1
6. docker 服务失败导致 telnet 不通，抓包表现为 宿主机收到了 telnet 的包，但不会转发给容器内部。
```bash
[root@localhost feiyang]# tcpdump -i any port 9200  -nnn
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked), capture size 262144 bytes
21:44:12.657584 IP 192.168.1.78.12118 > 192.168.1.89.9200: Flags [S], seq 2957511281, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
21:44:12.657654 IP 192.168.1.78.12118 > 172.17.0.2.9200: Flags [S], seq 2957511281, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
21:44:12.657658 IP 192.168.1.78.12118 > 172.17.0.2.9200: Flags [S], seq 2957511281, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
21:44:12.657711 IP 172.17.0.2.9200 > 192.168.1.78.12118: Flags [S.], seq 3836648310, ack 2957511282, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
21:44:12.657711 IP 172.17.0.2.9200 > 192.168.1.78.12118: Flags [S.], seq 3836648310, ack 2957511282, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
21:44:12.657727 IP 192.168.1.89.9200 > 192.168.1.78.12118: Flags [S.], seq 3836648310, ack 2957511282, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
21:44:12.657913 IP 192.168.1.78.12118 > 192.168.1.89.9200: Flags [.], ack 1, win 229, length 0
21:44:12.657921 IP 192.168.1.78.12118 > 172.17.0.2.9200: Flags [.], ack 1, win 229, length 0
21:44:12.657922 IP 192.168.1.78.12118 > 172.17.0.2.9200: Flags [.], ack 1, win 229, length 0
21:45:31.315560 IP 192.168.1.78.12118 > 192.168.1.89.9200: Flags [F.], seq 1, ack 1, win 229, length 0
21:45:31.315596 IP 192.168.1.78.12118 > 172.17.0.2.9200: Flags [F.], seq 1, ack 1, win 229, length 0
21:45:31.315598 IP 192.168.1.78.12118 > 172.17.0.2.9200: Flags [F.], seq 1, ack 1, win 229, length 0
21:45:31.315985 IP 172.17.0.2.9200 > 192.168.1.78.12118: Flags [F.], seq 1, ack 2, win 229, length 0
21:45:31.315985 IP 172.17.0.2.9200 > 192.168.1.78.12118: Flags [F.], seq 1, ack 2, win 229, length 0
21:45:31.316012 IP 192.168.1.89.9200 > 192.168.1.78.12118: Flags [F.], seq 1, ack 2, win 229, length 0
21:45:31.316225 IP 192.168.1.78.12118 > 192.168.1.89.9200: Flags [.], ack 2, win 229, length 0
21:45:31.316277 IP 192.168.1.78.12118 > 172.17.0.2.9200: Flags [.], ack 2, win 229, length 0
21:45:31.316279 IP 192.168.1.78.12118 > 172.17.0.2.9200: Flags [.], ack 2, win 229, length 0
```

# ubuntu network config
```
# /etc/network/interfaces

auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual
    bond-master bond0

auto eno2
iface eno2 inet manual
    bond-master bond0

auto bond0
iface bond0 inet manual
    bond-slaves eno1 eno2
    bond-mode 4
    bond-miimon 100
    bond-lacp-rate 1
    bond-xmit_hash_policy layer3+4

auto bond0.1000
iface bond0.1000 inet static
    address 10.1.1.10
    netmask 255.255.255.0
    gateway 10.1.1.254
    vlan-raw-device bond0

#----------------------------------
#ubuntu default
# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto ens33
iface ens33 inet dhcp
```