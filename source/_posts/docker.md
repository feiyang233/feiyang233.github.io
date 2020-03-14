---
title: docker 学习笔记
tags: 
  - docker 
  - kubernetes
categories: Linux
date: 2019-08-01 00:00:00
---
本文记录了一些基础的 docker 知识。
<!--more--> 
# reference
1. [你确定你会写 Dockerfile 吗](https://mp.weixin.qq.com/s/wNCfYERWU3GOBHI2juTpmg)
2. [Docker stop或者Docker kill为何不能停止容器](https://mp.weixin.qq.com/s/eZCi73pOFq0sSoMYSVujZw)
3. [P2P镜像分发Dragonfly使用](https://mp.weixin.qq.com/s/HwmBNPaWBmD3-VW5V04MLg)
4. [Docker 数据持久化的三种方案](https://mp.weixin.qq.com/s/5Nu9fnH7K-Y6a_pA2z2haA)
5. [Docker — 从入门到实践](https://yeasy.gitbooks.io/docker_practice/content/)
6. [docker-proxy 存在合理性分析](https://mp.weixin.qq.com/s/S4aMdmQW50HR7Lj2z2-VZQ)
7. [Docker 的原理](https://mp.weixin.qq.com/s/F8dFqdMBI0pfXJWow5dHhQ)
8. [Docker 存储驱动与磁盘文件系统](https://mp.weixin.qq.com/s/bRQUrpSbZyZ5kUg7vfvc1A) 内含格式化磁盘，挂载磁盘

# 安装 docker
分享一篇[CentOS 7 下 yum 方式安装 Docker 环境](https://mp.weixin.qq.com/s/BCzL57RmJja2gfZ67a2Z9Q)
```
sudo yum install -y yum-utils device-mapper-persistent-data lvm2

sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

sudo yum install -y docker-ce

sudo systemctl start docker
```

一个“容器”，实际上是一个由 Linux Namespace、Linux Cgroups 和 rootfs 三种技术构建出来的进程的隔离环境。
这三种技术介绍？
Namespace 的作用是“隔离”，它让应用进程只能看到该 Namespace 内的“世界”；而 Cgroups 的作用是“限制”

rootfs 只是一个操作系统所包含的文件、配置和目录，并不包括操作系统内核。在 Linux 操作系统中，这两部分是分开存放的，操作系统只有在开机启动时才会加载指定版本的内核镜像。正是由于 rootfs 的存在，容器才有了一个被反复宣传至今的重要特性：一致性。

docker commit，实际上就是在容器运行起来后，把最上层的“可读写层”，加上原先容器镜像的只读层，打包组成了一个新的镜像。当然，下面这些只读层在宿主机上是共享的，不会占用额外的空间。
# docker volume
镜像的各个层，保存在 /var/lib/docker/aufs/diff 目录下，在容器进程启动后，它们会被联合挂载在 /var/lib/docker/aufs/mnt/ 目录中，这样容器所需的 rootfs 就准备好了。  
volume /test 挂载出来之后，文件会出现在了宿主机上对应的临时目录里，但是如果你在宿主机上查看该容器的可读写层，虽然可以看到这个 /test 目录，但其内容是空的。
而由于使用了联合文件系统，你在容器里对镜像 rootfs 所做的任何修改，都会被操作系统先复制到这个可读写层，然后再修改。这就是所谓的：Copy-on-Write。  
有些时候，会由于配置文件的出错导致容器运行失败，这个时候不能进入容器修改文件，只能去 /var/lib/docker/overlay2/ID/diff 下面，找到对应的配置文件进行修改。


# docker 常用命令
```bash
# 启动 docker 服务 
systemctl start docker
systemctl enable docker

# 查看当前系统 Docker 信息 
docker info

# 拉取 docker 镜像 
docker pull image_name
# 从 Docker hub 上下载某个镜像 
docker pull centos:latest

# 查看宿主机上的镜像，Docker 镜像保存在 / var/lib/docker 目录下:
docker images

REPOSITORY                      TAG                 IMAGE ID            CREATED             SIZE
mysql                           5.7                 1b30b36ae96a        8 days ago          372MB
zabbix/zabbix-web-nginx-mysql   centos-4.0-latest   8be5f91b2fa1        3 weeks ago         415MB
zabbix/zabbix-server-mysql      centos-4.0-latest   8e5becf45c4e        3 weeks ago         326MB

# 删除镜像 
docker rmi image_name/image_id
docker rmi zabbix/zabbix-web-nginx-mysql:centos-4.0-latest 或者 docker rmi 8be5f91b2fa1

# 查看当前有哪些容器正在运行 
docker ps
# 查看所有容器 
docker ps -a

CONTAINER ID        IMAGE                                             COMMAND                  CREATED             STATUS                       PORTS                           NAMES
b30307ad65be        zabbix/zabbix-web-nginx-mysql:centos-4.0-latest   "docker-entrypoint.sh"   7 days ago          Exited (255) 8 minutes ago   443/tcp, 0.0.0.0:8080->80/tcp   zabbix-web-nginx-mysql
0ad822cd52b7        zabbix/zabbix-server-mysql:centos-4.0-latest      "docker-entrypoint.sh"   7 days ago          Exited (255) 8 minutes ago   0.0.0.0:10051->10051/tcp        zabbix-server-mysql
d01c89a112f7        mysql:5.7                                         "docker-entrypoint.s…"   7 days ago          Exited (255) 8 minutes ago   3306/tcp, 33060/tcp             mysql-server

# 启动、停止、重启容器命令：
docker start container_name/container_id
docker stop container_name/container_id
docker restart container_name/container_id

# 后台启动一个容器后，如果想进入到这个容器，可以使用 attach
docker attach container_name/container_id

# 删除容器 
docker rm container_name/container_id

# 查看容器日志 
docker logs -f container_name/container_id

# 查看容器 IP 地址 
docker inspect container_name/container_id

# 进入容器 
docker exec -it container_name/container_id bash

# 从 Docker 容器与宿主机相互传输文件 
[root@localhost tmp]# docker cp --help

Usage:	docker cp [OPTIONS] CONTAINER:SRC_PATH DEST_PATH|-
	docker cp [OPTIONS] SRC_PATH|- CONTAINER:DEST_PATH

Copy files/folders between a container and the local filesystem

docker cp zabbix_config.sql mysql-server:/tmp
docker cp mysql-server:/tmp/zabbix_config.sql /tmp


# 批量删除所有已经退出的容器 
docker rm -v $(docker ps -aq -f status=exited)
# 批量删除所有仅创建并未成功运行的容器 
docker rm -v $(docker ps -aq -f status=created)
```

# docker logs 
[删除 logs](https://blog.csdn.net/yjk13703623757/article/details/80283729) 设置 logs [Daemon configuration file](https://docs.docker.com/engine/reference/commandline/dockerd/)

# docker iptables 
docker 默认会修改 iptables，但有时候会造成 iptables ip 限制失效的问题。如果对 iptables 比较了解的同学，可以 [Prevent Docker from manipulating iptables](https://docs.docker.com/network/iptables/)  

```bash
# vim /etc/docker/daemon.json
{
  "iptables": false,
  "log-driver":"json-file",
  "log-opts": {"max-size":"500m", "max-file":"3"}
}

max-size=500m，意味着一个容器日志大小上限是500M，
max-file=3，意味着一个容器有三个日志，分别是id+.json、id+1.json、id+2.json。

systemctl daemon-reload

systemctl restart docker
```

还有一种办法就是添加 [DOCKER-USER](https://docs.docker.com/network/iptables/)  If you need to add rules which load before Docker’s rules, add them to the DOCKER-USER chain. These rules are loaded before any rules Docker creates automatically.
```bash
*filter
:INPUT DROP [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]

:DOCKER-USER - [0:0]
-A FORWARD -j DOCKER-USER
-A DOCKER-USER -s x.x.x.x -p tcp --dport 80 -j RETURN
-A DOCKER-USER -p tcp --dport 80 -j DROP
-A DOCKER-USER -p udp --dport 80 -j DROP
-A DOCKER-USER -j RETURN

COMMIT
```
# Dockerfile
首先，安利一下 [Dockerfile 优化指南](https://mp.weixin.qq.com/s/wNCfYERWU3GOBHI2juTpmg)

```
FROM python:3.7

RUN apt-get update \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /usr/src/app
COPY requirements.txt ./
RUN pip install -r requirements.txt
COPY . .

EXPOSE 8000
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]

```

# tag
docker 镜像有许多的版本，不同版本间差异需要注意。举个例子，我遇到因为版本不同，ip route get 结尾多了一个 uid 0
```bash
#执行这个命令，在不同 tag 版本的 Python 镜像里，结果会不一样。
ip -4 route get 8.8.8.8

# 结果多了一个 uid 会导致提取 IP 时出错。
8.8.8.8 via 192.168.0.1 dev eth0 src 192.168.0.105 uid 0 

8.8.8.8 via 192.168.0.1 dev eth0 src 192.168.0.105 

# 两者之间的差异是因为 iproute2 版本不一样

root@feiy:/etc/apt# apt-cache policy iproute2
iproute2:
  Installed: 4.9.0-1+deb9u1
  Candidate: 4.9.0-1+deb9u1
  Version table:
 *** 4.9.0-1+deb9u1 500
        500 http://deb.debian.org/debian stretch/main amd64 Packages
        100 /var/lib/dpkg/status


root@feiy:/etc/apt# apt-cache policy iproute2
iproute2:
  Installed: 4.20.0-2
  Candidate: 4.20.0-2
  Version table:
 *** 4.20.0-2 500
        500 http://deb.debian.org/debian buster/main amd64 Packages
        100 /var/lib/dpkg/status

```
# network
docker 是基于 iptables 进行流量的转发，添加了一个虚拟的网卡 docker0 。 如果想从容器内部访问宿主机的 IP，比如从 172.17.0.3 访问宿主机的内网地址 192.168.1.10 ， 我们从 tcpdump 在网卡 docker0  抓包可以看到流量。 但是在内网网卡抓包没有结果。猜测原因是：当数据包从网卡 docker0 转发后，直接在内核进入 INPUT 链。所以，如果想访问宿主机的 IP， 需要在 INPUT 添加相关规则，允许访问。  iptables -A INPUT -i docker0 -j ACCEPT

# mac 上使用 docker
1. volume 的位置和 linux 不同，解决办法如下

```bash
cd ~/Library/Containers/com.docker.docker/Data/vms/0/

# 连接进入 tty
screen tty

# 在 tty 可以就可以找到 docker container volume

cd /var/lib/docker/volumes/
```

2. 如果想从容器内部访问 mac，也和 Linux 不同。详情可以看官网[docker mac network](https://docs.docker.com/docker-for-mac/networking/) 在容器内部可以使用 host.docker.internal 就可以访问到 Mac 宿主机啦。 而在 Linux 机器上, 容器可以直接使用宿主机的 IP 来访问宿主机

# docker-proxy
can refer this (article)[https://mp.weixin.qq.com/s/S4aMdmQW50HR7Lj2z2-VZQ]
```bash
# /etc/docker/daemon.json
# disable docker-proxy
“userland-proxy”:false 
```

# find process 
In one hosts, if we have many dockers, sometimes we need find some proceess belong to which docker.
```bash
pstree -laps [pid]

pstree -laps 38564
systemd,1
  `-dockerd,171350
      `-docker-containe,171362 --config /var/run/docker/containerd/containerd.toml
          `-docker-containe,332151 -namespace moby -workdir /var/lib/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/38dfefa95c1f8b4827e0a95834119b3756775d389820ea90d95878d3518eed67 -address /var/run/docker/containerd/docker-containerd.sock -containerd-binary /usr/bin/docker-containerd -runtime-root /var/run/docker/runtime-runc
              `-smb,332174 /usr/local/bin/smb run -e live -c id -i sg
                  `-sh,332221 -c gunicorn webapi.wsgi -c webapi/gunicorn_config.py
                      `-gunicorn: maste,332306                                                  
                          `-gunicorn: worke,38564                                                  
                              |-{gunicorn: worke},38613
                              |-{gunicorn: worke},83641
                              |-{gunicorn: worke},83642
                              |-{gunicorn: worke},83643
                              |-{gunicorn: worke},83644
                              |-{gunicorn: worke},83645
                              |-{gunicorn: worke},83646
                              `-{gunicorn: worke},83647

# 38dfefa95c1f8b4827e0a95834119b3756775d389820ea90d95878d3518eed67 is container ID
```

# docker-compose

https://wiki.jikexueyuan.com/project/docker-technology-and-combat/commands.html