---
title: 运维中踩过的坑
date: 2019-07-05 10:58:36
tags:
  - docker
  - elk
  - jenkins
  - python
categories: Linux
---
仅以本文记录那些年弟弟背过的锅！
<!--more-->  
# iptables
1. 大部分时候，iptables 只存了 filter 表。 对于 nat 表，我们一旦 restart iptables， nat 表的规则就会被刷新。建议用 reload
2. -A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT 这一条 iptables 的规则也非常重要。Packets in a RELATED or ESTABLISHED state are those ones which belong to an already opened connection; you'll generally want to accept them, otherwise connections will get established correctly but nothing will be able to flow after the initial handshake. 如果没有这一条，会遇到 DNS 解析失败， curl 失败。 凡是 iptables 没有允许的 IP, 都不能正常的工作。 例如 DNS 查询发包后，三次握手建立。回包收到了，却会被 iptables 阻挡，上层应用无法拿到解析的结果，导致 hang 住。
3. tcpdump 抓包分析时， 进入的包都可以抓到，不会受到 iptables, 发出的包会受到 iptables 影响，可能被 iptables 阻挡导致抓包失败。
4. 如果是命令行添加的规则，例如 iptables -t nat -A OUTPUT -s 172.17.0.3 -p tcp --dport 10050 -j ACCEPT  在使用 reload 后，其规则一样会被刷掉。但 docker 的规则不会被 reload 刷掉，会被 restart 刷掉。这一点有疑问，期待大神给弟弟解惑。
5. docker iptables filter 表 input 链规则失效，原因是：从 NAT 表 prerouting 链匹配到 DOCKER 链，直接与容器沟通，跳过了 input 链。感谢智凯哥哥的解决办法如下：
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

# python
1. tab 键与空格不能混用
2. 函数不要嵌套，例如 result=A(B(C())) 这样不利于 debug，检查每个函数的返回值
3. 在 multiprocessing 多进程中 apply_async()  报错不容易定位，最好使用 try catch 把报错写成 log


# jenkins
1. 在部署的时候，ansible 一直在 gathering facts 卡住了，直到 timeout。 [网友解答](https://serverfault.com/questions/630253/ansible-stuck-on-gathering-facts) 我遇到的是 [control_path](https://docs.ansible.com/ansible/2.4/intro_configuration.html#control-path) 文件太多了，导致了 jenkins deploy node 卡住，任务无法进行。 需要删除该部署节点下面的 control_path_dir 的文件，清空。


# docker
1. docker container IP default  is 172.17.0.0/16  检查 iptables 是否阻挡
2. docker -v 挂载出来的时候，要注意文件夹权限问题。
3. docker logs 日志文件很大的时候，记得删除。[Docker容器日志查看与清理](https://blog.csdn.net/yjk13703623757/article/details/80283729) 也可以在 /etc/docker/daemon 里[设置 log 的大小](https://feiyang233.club/post/docker/#docker-iptables)。

# NetworkManager
1. 报错 bus-manager: could not create org.freedesktop.DBus proxy 直接 stop NetworkManager 就行了。

# 文件权限问题
1. /tmp permission 又搞我 drwxrwxrwt.
2. 对于目录文件来说，可读表示能够读取目录内的文件列表；可写表示能够在目录内新增、删除、重命名文件；可执行表示能够进入该目录。
3. file and directory [permission](https://blog.csdn.net/NOT_GUY/article/details/63685650)
4. [atime, ctime and mtime in Unix filesystems](https://www.unixtutorial.org/atime-ctime-mtime-in-unix-filesystems)

# DNS
1. /etc/resolv.conf 有两个默认的值至关重要，一个是超时的 timeout，一个是重试的 attempts，默认情况下，前者是 5s 后者是 2 次。对于日常的应用来说，包括 web server、mail client、db 以及各种 app server 等等，任何使用 glibc resolver 都需要经过 resolv.conf 文件。对于 libresolv 来说，只认 resolv.conf 的前三个 nameserver，所以写的再多也没什么意义。正常情况下，resolver 会从上至下进行解析，每个 nameserver 等待 timeout 的时间，如果一直到第三个都没结果，resolver 会重复上面的步骤 (attempts – 1) 次。

# Ansible
1. user 模块，密码必须要加密。需要用到 [Ansible ad-hoc command](https://docs.ansible.com/ansible/latest/reference_appendices/faq.html#how-do-i-generate-encrypted-passwords-for-the-user-module)


# Markdown
1. Markdown链接括号的问题: %28 代替(, %29代替) 主要是后者会歧义链接部分的结束. 这是使用url符号码去代替ascii的符号. 能够解决这个问题


# Django
1. 文件名第一个字符为空格，设置 static 时，多了一个空格 %20 ，导致资源路径出出错。
2. Mac Django 连接 MySQL, [官网](https://docs.djangoproject.com/en/2.2/ref/databases/#mysql-notes)上提供了两种方法。第一种 mysqlclient 在 Mac 上有遇到错误。 [mysqlclient 官网](https://pypi.org/project/mysqlclient/)也有说明, [Google一下](https://ruddra.com/posts/install-mysqlclient-macos/)后发现是因为我的 Mac 没有安装 clang，安装完 xcode-select --install 再 pip install mysqlclient 就可以了。  
第二种，安装[Connector](https://dev.mysql.com/downloads/connector/python/) 在 setting.py 里添加
```python
DATABASES = {
    'default': {
        'NAME': 'user_data',
        'ENGINE': 'mysql.connector.django',
        'USER': 'mysql_user',
        'PASSWORD': 'password',
        'OPTIONS': {
          'autocommit': True,
        },
    }
}
```
# Linux
1. 内存泄漏指由于疏忽或错误造成程序未能释放已经不再使用的内存。内存泄漏并非指内存在物理上的消失，而是应用程序分配某段内存后，由于设计错误，导致在释放该段内存之前就失去了对该段内存的控制，从而造成了内存的浪费。 有一个服务跑在容器里面，当连接的时候总是 time out， 查看日志 是 stream is closed，还有一个错误是 No such file or directory: u'/var/run/tmp/summary_398320.db'  当我们进入 /var/run/tmp 发现有太多的文件，每一个进程都会有一个 pid 命名的文件。说明 worker 进程不断被 kill 然后又被 master 拉起，当我们到宿主机上看 docker stats ， 发现该容器的内存已经满了，导致了 worker 进程不断被杀。需要增大容器的 memory 初始值。补充（gunicorn 会启动一组 worker进程，所有worker进程公用一组listener，在每个worker中为每个listener建立一个wsgi server。每当有HTTP链接到来时，wsgi server创建一个协程来处理该链接，协程处理该链接的时候，先初始化WSGI环境，然后调用用户提供的app对象去处理HTTP请求。）
2. [xargs与管道的区别](https://www.cnblogs.com/wangqiguo/p/6464234.html) 其实基本上linux的命令中很多的命令的设计是先从命令行参数中获取参数，然后从标准输入中读取。另外很多程序是不处理标准输入的，例如 kill , rm 这些程序如果命令行参数中没有指定要处理的内容则不会默认从标准输入中读取。所以 xargs 可以用来传递参数

# 读取超大日志文件
如果日志没有做切割，有可能会导致生成一个超大的日志，这个时候对应我们查看日志文件非常的不方便。这个时候有一个方法就是用 `split` 将大日志均分为小日志
```bash

#按行数均分
split -l 10000 test.log -d -a 4 test_

#-l, --lines=NUMBER：对file进行切分，每个文件有NUMBER行。
#每个文件10000行(-l 10000)
#文件名称后缀系数不是字母而是数字（-d）
#后缀系数为四位数（-a 4）

# 也可以按文件大小均分
split -b test.log -d -a 4 test_

#-b, --bytes=SIZE：对file进行切分，每个小文件大小为SIZE。可以指定单位b,k,m,g。
```
# kafka
1. kafka listen all interfaces
In order to listen all the interfaces, comment  #host.name in /etc/kafka/server.properties     
This action can make Kafka listen in all interfaces, but it will change hostname to localhost, then Kafka cannot communicate with each other. 
Error is the connection with node -1 failed.
```bash
Info in zookeeper: "PLAINTEXT://localhost:29092"],"jmx_port":9093,"host":"localhost",
```
solution: Kafka can listen in all interfaces and connection with each other.
```
listeners=PLAINTEXT://0.0.0.0:29092
advertised.host.name=10.66.236.43
host.name=10.66.236.43
```