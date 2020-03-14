---
title: zabbix 从入门到放弃
date: 2019-06-06 11:00:56
tags:
  - docker
  - zabbix
categories: Linux
---
docker 安装 zabbix, 添加主机，设置报警，性能调优。
<!--more-->
# 监控模式
zabbix 有两种监控模式，主动和被动。在客户端与服务端之间还可以加一个 proxy，入下图所示  
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20190823152533.png)

**需要注意 iptables问题**: 跨地区监控的时候， proxy 必须监听在所有网卡上。内网是为了和客户端通信，外网是为了和服务端通信。我曾试过 proxy 只监听在内网，因为是主动模式，层层上报信息，在 zabbix server 还是能发现 proxy 的存活。但是当我打算添加一台 host 时，却一直报错。原因就是 proxy 和 服务端是外网通信的，proxy 发包查询 host 的信息（监控项等），因为只监听内网，服务端的回包 proxy 无法获取，导致通信失败。
# LAMP 架构安装
基于官方的 LAMP 架构，按照最简单的原生方式来部署，不做任何多余优化。
```
# 安装必要依赖包 
yum install -y httpd mariadb-server mariadb php php-mysql php-gd libjpeg* php-ldap php-odbc php-pear php-xml php-xmlrpc php-mhash

# 修改 apache 配置 
vim /etc/httpd/conf/httpd.conf
ServerName 192.168.1.10:8080
Listen 192.168.1.10:8080
DirectoryIndex index.html index.php

# 修改 php 时区 
vim /etc/php.ini
date.timezone = Asia/Singapore

# 启动 httpd 服务 
systemctl start httpd.service

# 修改数据库存储的位置 /data
vim /etc/my.cnf
[mysqld]
datadir=/data/mysql
socket=/data/mysql/mysql.sock
bind-address = 127.0.0.1
max_connections = 1000

# 最大连接数很关键，如果 zabbix-server StartPollers= 设置过大
# 则很容易出现，connection to database 'zabbix' failed: [1040] Too many connections

[client]
port=3306
socket=/data/mysql/mysql.sock

# 启动 mariadb 服务 
systemctl start mariadb.service

# 初始化 mysql 数据库，并配置 root 用户密码 
mysql_secure_installation

# 万一新版本忘记随机密码可以通过日志获取 
grep 'temporary password' /var/log/mysqld.log

# 创建一个测试页，测试 LAMP 是否搭建成功 
cat > /var/www/html/index.php << EOF
<?php
phpinfo();
?>
EOF

# 创建 zabbix 数据库 
mysql -uroot -p

mysql> create database zabbix character set utf8 collate utf8_bin;
mysql> grant all privileges on zabbix.* to zabbix@localhost identified by 'zabbix';
mysql> quit;

# 部署 zabbix
rpm -Uvh https://repo.zabbix.com/zabbix/4.2/rhel/7/x86_64/zabbix-release-4.2-2.el7.noarch.rpm
yum clean all
yum install -y zabbix-server-mysql zabbix-web-mysql zabbix-agent
zcat /usr/share/doc/zabbix-server-mysql*/create.sql.gz | mysql -uzabbix -p zabbix
# 输入密码 zabbix

# 配置数据库用户及密码 
vim /etc/zabbix/zabbix_server.conf
DBPassword=zabbix
DBSocket=/data/mysql/mysql.sock

# 修改时区 
vim /etc/httpd/conf.d/zabbix.conf
php_value date.timezone Asia/Singapore

# 启动 zabbix 并设置自启动服务 
systemctl restart zabbix-server zabbix-agent httpd
systemctl enable zabbix-server zabbix-agent httpd mariadb
```
一切就绪，打开浏览器，输入 http://ServerName:port/zabbix
# zabbix_server.conf 参数调优
```
CacheSize=8G               # Host 过多时，需要增大 CacheSize
TrendCacheSize=2G          # __zbx_mem_realloc(): out of memory (requested 108424 bytes) 
Timeout=30                 # __zbx_mem_realloc(): please increase CacheSize configuration parameter
StartPollers=500           # Poller 会导致连接数增大。需要调大数据库的最大连接数
StartPollersUnreachable=100
HousekeepingFrequency=0
```
# docker 搭建
```bash
# install docker-ce
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce
sudo systemctl start docker

# 做数据映射后的方案 
mkdir -p /data/docker/mysql/zabbix/data
mkdir -p /data/docker/zabbix/alertscripts
mkdir -p /data/docker/zabbix/externalscripts
```

然后是安装 zabbix 前端，后端，数据库。
```bash
# 数据库。
docker run --name mysql-server -t \
-e MYSQL_DATABASE="zabbix" \
-e MYSQL_USER="zabbix" \
-e MYSQL_PASSWORD="feiyang@2019+" \
-e MYSQL_ROOT_PASSWORD="feiyang@2019+" \
-v /data/zabbix_data:/var/lib/mysql  \
-d mysql:5.7  \
--character-set-server=utf8 --collation-server=utf8_bin

```

```bash
# 后端 参数已经调优
docker run --name zabbix-server-mysql  \
-e DB_SERVER_HOST="mysql-server"  \
-e MYSQL_DATABASE="zabbix"  \
-e MYSQL_USER="zabbix"  \
-e MYSQL_PASSWORD="feiyang@2019+"  \
-e MYSQL_ROOT_PASSWORD="feiyang@2019+"  \
-e ZBX_TIMEOUT=30   \
-e ZBX_CACHESIZE=8G  \
-e ZBX_TRENDCACHESIZE=2G  \
-e ZBX_STARTPOLLERS=500   \
-e ZBX_STARTPOLLERSUNREACHABLE=100  \
-e ZBX_HOUSEKEEPINGFREQUENCY=0  \
-v /data/zabbix/alertscripts:/usr/lib/zabbix/alertscripts  \
-v /data/zabbix/externalscripts:/usr/lib/zabbix/externalscripts  \
-v /data/zabbix/conf:/etc/zabbix  \
--link mysql-server:mysql  \
-p 10051:10051  \
-d zabbix/zabbix-server-mysql:centos-4.2-latest

```

```bash
# 前端
docker run --name zabbix-web-nginx-mysql \
-e DB_SERVER_HOST="mysql-server" \
-e MYSQL_DATABASE="zabbix" \
-e MYSQL_USER="zabbix" \
-e MYSQL_PASSWORD="feiyang@2019+" \
-e MYSQL_ROOT_PASSWORD="feiyang@2019+" \
-e PHP_TZ="Asia/Singapore" \
--link mysql-server:mysql \
--link zabbix-server-mysql:zabbix-server \
-p 8080:80 \
-d zabbix/zabbix-web-nginx-mysql:centos-4.2-latest
```

安装完成后，在浏览器打开 http://localhost:8080  默认的账户是 `Admin`  密码是 `zabbix`

# ansible 批量添加主机 

```yml
---
  - name: add zabbix hosts
    local_action:
      module: zabbix_host
      server_url: "{{ var_server_url }}"
      login_user: "{{ var_login_user }}"
      login_password: "{{ var_login_password }}"
      host_name: "{{ inventory_hostname }}"
      visible_name: "{{ inventory_hostname }}-{{function}}"
      host_groups:
        - "{{ var_host_group }}"
      link_templates:
        - Template Sea Ops OS Linux
        - Template Sea Ops Disk IO Linux
      #status: disabled
      status: enabled
      state: present
      interfaces:
      - type: 1
        main: 1
        useip: 1
        ip: "{{  var_lanip | default(inventory_hostname) }}"
        dns: ""
        port: 10050
```


# Action
设置触发警告的 Action 时，当 Step 设置为从 1 到 0 时，会一直发送告警信息，直到事件状态变成 OK，当 Step 设置为从 1 到 1 时，则只会发送一次告警，后面不会继续发送告警信息。


# Zabbix 监控
## 监控网页状态
zabbix 自带的 Web monitoring 就可以进行简单的网页监控。目前官方的 zabbix 版本是 4.2 此时日期 2019-07-09  
首先是找到一台机器 Go to Configuration → Hosts, pick a host and click on Web in the row of that host. Then click on Create web scenario. 详情请看[官方文档](https://www.zabbix.com/documentation/4.2/manual/web_monitoring/example)，然后是添加报警，[网页监控的官方文档](https://www.zabbix.com/documentation/4.2/manual/web_monitoring/items)也是介绍的非常详细。  
具体的监控图表信息，可以在 zabbix 主页的 Monitoring -> Web 可以看到网页监控的详细信息。

## 监控 DNS 
[官方文档 4.2 版本](https://www.zabbix.com/documentation/4.2/manual/config/items/itemtypes/zabbix_agent?s[]=net&s[]=dns&s[]=record)  
zabbix默认支持检查解析成功与否和具体的解析结果。对应内置的KEY
```
net.dns[<ip>,zone,<type>,<timeout>,<count>]
net.dns.record[<ip>,zone,<type>,<timeout>,<count>]
ip 指DNS服务器地址。
zone 指要解析的域名
type 指解析的记录类型
timeout 指超时时间 默认1 秒
count 指解析失败重试的次数 默认 2次

trigger {host:net.dns[dns_server,domain,A,1,2].count(#3)}=0
```

# 数据库表优化
```sql
DELIMITER $$
CREATE PROCEDURE `partition_create`(SCHEMANAME varchar(64), TABLENAME varchar(64), PARTITIONNAME varchar(64), CLOCK int)
BEGIN
        /*
           SCHEMANAME = The DB schema in which to make changes
           TABLENAME = The table with partitions to potentially delete
           PARTITIONNAME = The name of the partition to create
        */
        /*
           Verify that the partition does not already exist
        */

        DECLARE RETROWS INT;
        SELECT COUNT(1) INTO RETROWS
        FROM information_schema.partitions
        WHERE table_schema = SCHEMANAME AND table_name = TABLENAME AND partition_description >= CLOCK;

        IF RETROWS = 0 THEN
                /*
                   1. Print a message indicating that a partition was created.
                   2. Create the SQL to create the partition.
                   3. Execute the SQL from #2.
                */
                SELECT CONCAT( "partition_create(", SCHEMANAME, ",", TABLENAME, ",", PARTITIONNAME, ",", CLOCK, ")" ) AS msg;
                SET @sql = CONCAT( 'ALTER TABLE ', SCHEMANAME, '.', TABLENAME, ' ADD PARTITION (PARTITION ', PARTITIONNAME, ' VALUES LESS THAN (', CLOCK, '));' );
                PREPARE STMT FROM @sql;
                EXECUTE STMT;
                DEALLOCATE PREPARE STMT;
        END IF;
END$$
DELIMITER ;
DELIMITER $$
CREATE PROCEDURE `partition_drop`(SCHEMANAME VARCHAR(64), TABLENAME VARCHAR(64), DELETE_BELOW_PARTITION_DATE BIGINT)
BEGIN
        /*
           SCHEMANAME = The DB schema in which to make changes
           TABLENAME = The table with partitions to potentially delete
           DELETE_BELOW_PARTITION_DATE = Delete any partitions with names that are dates older than this one (yyyy-mm-dd)
        */
        DECLARE done INT DEFAULT FALSE;
        DECLARE drop_part_name VARCHAR(16);

        /*
           Get a list of all the partitions that are older than the date
           in DELETE_BELOW_PARTITION_DATE.  All partitions are prefixed with
           a "p", so use SUBSTRING TO get rid of that character.
        */
        DECLARE myCursor CURSOR FOR
                SELECT partition_name
                FROM information_schema.partitions
                WHERE table_schema = SCHEMANAME AND table_name = TABLENAME AND CAST(SUBSTRING(partition_name FROM 2) AS UNSIGNED) < DELETE_BELOW_PARTITION_DATE;
        DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;

        /*
           Create the basics for when we need to drop the partition.  Also, create
           @drop_partitions to hold a comma-delimited list of all partitions that
           should be deleted.
        */
        SET @alter_header = CONCAT("ALTER TABLE ", SCHEMANAME, ".", TABLENAME, " DROP PARTITION ");
        SET @drop_partitions = "";

        /*
           Start looping through all the partitions that are too old.
        */
        OPEN myCursor;
        read_loop: LOOP
                FETCH myCursor INTO drop_part_name;
                IF done THEN
                        LEAVE read_loop;
                END IF;
                SET @drop_partitions = IF(@drop_partitions = "", drop_part_name, CONCAT(@drop_partitions, ",", drop_part_name));
        END LOOP;
        IF @drop_partitions != "" THEN
                /*
                   1. Build the SQL to drop all the necessary partitions.
                   2. Run the SQL to drop the partitions.
                   3. Print out the table partitions that were deleted.
                */
                SET @full_sql = CONCAT(@alter_header, @drop_partitions, ";");
                PREPARE STMT FROM @full_sql;
                EXECUTE STMT;
                DEALLOCATE PREPARE STMT;

                SELECT CONCAT(SCHEMANAME, ".", TABLENAME) AS `table`, @drop_partitions AS `partitions_deleted`;
        ELSE
                /*
                   No partitions are being deleted, so print out "N/A" (Not applicable) to indicate
                   that no changes were made.
                */
                SELECT CONCAT(SCHEMANAME, ".", TABLENAME) AS `table`, "N/A" AS `partitions_deleted`;
        END IF;
END$$
DELIMITER ;
DELIMITER $$
CREATE PROCEDURE `partition_maintenance`(SCHEMA_NAME VARCHAR(32), TABLE_NAME VARCHAR(32), KEEP_DATA_DAYS INT, HOURLY_INTERVAL INT, CREATE_NEXT_INTERVALS INT)
BEGIN
        DECLARE OLDER_THAN_PARTITION_DATE VARCHAR(16);
        DECLARE PARTITION_NAME VARCHAR(16);
        DECLARE OLD_PARTITION_NAME VARCHAR(16);
        DECLARE LESS_THAN_TIMESTAMP INT;
        DECLARE CUR_TIME INT;

        CALL partition_verify(SCHEMA_NAME, TABLE_NAME, HOURLY_INTERVAL);
        SET CUR_TIME = UNIX_TIMESTAMP(DATE_FORMAT(NOW(), '%Y-%m-%d 00:00:00'));

        SET @__interval = 1;
        create_loop: LOOP
                IF @__interval > CREATE_NEXT_INTERVALS THEN
                        LEAVE create_loop;
                END IF;

                SET LESS_THAN_TIMESTAMP = CUR_TIME + (HOURLY_INTERVAL * @__interval * 3600);
                SET PARTITION_NAME = FROM_UNIXTIME(CUR_TIME + HOURLY_INTERVAL * (@__interval - 1) * 3600, 'p%Y%m%d%H00');
                IF(PARTITION_NAME != OLD_PARTITION_NAME) THEN
                        CALL partition_create(SCHEMA_NAME, TABLE_NAME, PARTITION_NAME, LESS_THAN_TIMESTAMP);
                END IF;
                SET @__interval=@__interval+1;
                SET OLD_PARTITION_NAME = PARTITION_NAME;
        END LOOP;

        SET OLDER_THAN_PARTITION_DATE=DATE_FORMAT(DATE_SUB(NOW(), INTERVAL KEEP_DATA_DAYS DAY), '%Y%m%d0000');
        CALL partition_drop(SCHEMA_NAME, TABLE_NAME, OLDER_THAN_PARTITION_DATE);

END$$
DELIMITER ;
DELIMITER $$
CREATE PROCEDURE `partition_verify`(SCHEMANAME VARCHAR(64), TABLENAME VARCHAR(64), HOURLYINTERVAL INT(11))
BEGIN
        DECLARE PARTITION_NAME VARCHAR(16);
        DECLARE RETROWS INT(11);
        DECLARE FUTURE_TIMESTAMP TIMESTAMP;

        /*
         * Check if any partitions exist for the given SCHEMANAME.TABLENAME.
         */
        SELECT COUNT(1) INTO RETROWS
        FROM information_schema.partitions
        WHERE table_schema = SCHEMANAME AND table_name = TABLENAME AND partition_name IS NULL;

        /*
         * If partitions do not exist, go ahead and partition the table
         */
        IF RETROWS = 1 THEN
                /*
                 * Take the current date at 00:00:00 and add HOURLYINTERVAL to it.  This is the timestamp below which we will store values.
                 * We begin partitioning based on the beginning of a day.  This is because we don't want to generate a random partition
                 * that won't necessarily fall in line with the desired partition naming (ie: if the hour interval is 24 hours, we could
                 * end up creating a partition now named "p201403270600" when all other partitions will be like "p201403280000").
                 */
                SET FUTURE_TIMESTAMP = TIMESTAMPADD(HOUR, HOURLYINTERVAL, CONCAT(CURDATE(), " ", '00:00:00'));
                SET PARTITION_NAME = DATE_FORMAT(CURDATE(), 'p%Y%m%d%H00');

                -- Create the partitioning query
                SET @__PARTITION_SQL = CONCAT("ALTER TABLE ", SCHEMANAME, ".", TABLENAME, " PARTITION BY RANGE(`clock`)");
                SET @__PARTITION_SQL = CONCAT(@__PARTITION_SQL, "(PARTITION ", PARTITION_NAME, " VALUES LESS THAN (", UNIX_TIMESTAMP(FUTURE_TIMESTAMP), "));");

                -- Run the partitioning query
                PREPARE STMT FROM @__PARTITION_SQL;
                EXECUTE STMT;
                DEALLOCATE PREPARE STMT;
        END IF;
END$$
DELIMITER ;

DELIMITER $$
CREATE PROCEDURE`partition_maintenance_all`(SCHEMA_NAME VARCHAR(32))
BEGIN
               CALL partition_maintenance(SCHEMA_NAME, 'history', 30, 24, 14);
               CALL partition_maintenance(SCHEMA_NAME, 'history_log', 30, 24, 14);
               CALL partition_maintenance(SCHEMA_NAME, 'history_str', 30, 24, 14);
               CALL partition_maintenance(SCHEMA_NAME, 'history_text', 30, 24, 14);
               CALL partition_maintenance(SCHEMA_NAME, 'history_uint', 30, 24, 14);
               CALL partition_maintenance(SCHEMA_NAME, 'trends', 120, 24, 14);
               CALL partition_maintenance(SCHEMA_NAME, 'trends_uint', 120, 24, 14);
END$$
DELIMITER ;
```
Trends 120,(‘history’, 30, 24, 14), 最多保存 30 天的数据，每隔 24 小时生成一个分区，每次生成 14 个分区   

首先进入容器内部，将上面这个  partition.sql 导入数据库 mysql 
```bash
mysql  -uzabbix  -pfeiyang@2019+  zabbix  < partition.sql

# 在 mysql 容器内部 vim /opt/mysql.sh

#!bin/bash
mysql  -uzabbix -pfeiyang@2019+ zabbix -e"CALL partition_maintenance_all('zabbix')" 

chmod 755 /opt/mysql.sh
```
退出容器，在宿主机上，建立定时任务

```bash
# vim /etc/crontab

23 03  * * * root /bin/docker exec [mysql 容器 ID] bash -c  "cd /opt && bash mysql.sh" 

systemctl restart crond
```


# Zabbix api
[官方文档](https://www.zabbix.com/documentation/current/manual/api)

```python
# auth_zabbix

import requests
import json

url = 'http://IP:port/api_jsonrpc.php'  #docker 方式
# 非 docker 方式为 "http://IP:port/zabbix/api_jsonrpc.php"

post_data = {
    "jsonrpc": "2.0",
    "method": "user.login",
    "params": {
        "user": "xxx",
        "password": "xxx"
    },
    "id": 1,  
    
}
post_header = {'Content-Type': 'application/json'}

ret = requests.post(url, data=json.dumps(post_data), headers=post_header)

#print(ret)

zabbix_ret = json.loads(ret.text)

if not zabbix_ret.has_key('result'):
    print 'login error'
else:
    print zabbix_ret.get('result')
```

```python
# get hostid

import requests
import json

url = 'http://IP:port/api_jsonrpc.php'

server_list=["1.1.1.1","233.233.233.233"]

post_data = {

    "jsonrpc": "2.0",
    "method": "host.get",
    "params": {
    "filter": {
            "host": server_list
        },
    "sortfield": "host",
    },

    "id": 1,
    "auth": "由上文中的 auth_zabbix.py 得出"
}

post_header = {'Content-Type': 'application/json'}

ret = requests.post(url, data=json.dumps(post_data), headers=post_header)

zabbix_ret = json.loads(ret.text)

print zabbix_ret

if not zabbix_ret.has_key('result'):
    print 'login error'
else:
    print zabbix_ret.get('result')
    
hostid_list=[]
for i in zabbix_ret.get('result'):
	hostid_list.append(str(i['hostid']))

print hostid_list
```

```python
# get hist_data
import requests
import json
import time
import datetime

today = datetime.date.today()
today_unix = int(time.mktime(today.timetuple()))
tomorrow = today+datetime.timedelta(days=1)
tomorrow_unix = int(time.mktime(tomorrow.timetuple()))

print today_unix
print tomorrow_unix

url = 'http://IP:port/api_jsonrpc.php'

post_data = {

    "jsonrpc": "2.0",
    "method": "history.get",
    "params": {
        "output": "extend",
        "history": 3, # 0,1,2,3,4 History object types
        "itemids": "31023",
        "sortfield": "clock",
        "sortorder": "DESC",
        "time_from": "today_unix",
        "time_till": "tomorrow_unix"
       
    },

    "auth": "由上文中的 auth_zabbix.py 得出",
    "id": 1
}

post_header = {'Content-Type': 'application/json'}

ret = requests.post(url, data=json.dumps(post_data), headers=post_header)

zabbix_ret = json.loads(ret.text)

print zabbix_ret.get('result')

```

```python
# get trend_data
import requests
import json
import time
import datetime

today = datetime.date.today()
today_unix = int(time.mktime(today.timetuple()))
tomorrow = today+datetime.timedelta(days=1)
tomorrow_unix = int(time.mktime(tomorrow.timetuple()))


url = 'http://IP:port/api_jsonrpc.php'

post_data = {
    "jsonrpc": "2.0",
    "method": "trend.get",
    "params": {
        "output": [  # 定义 output 格式
            "itemid",
            "clock", # 当前时间
            "num",   # trend 一小时采集次数
            "value_min",
            "value_avg",
            "value_max"
        ],
        
        "itemids": [
            "28959",
            "28972"
            
        ],
        
        "time_from": today_unix,
        "time_till": tomorrow_unix
    },
    "auth": "由上文中的 auth_zabbix.py 得出",
    "id": 1
}

post_header = {'Content-Type': 'application/json'}

ret = requests.post(url, data=json.dumps(post_data), headers=post_header)

zabbix_ret = json.loads(ret.text)

print (zabbix_ret.get('result'))
```



# zabbix_get
从 server 端检测到 client 端的网络是否通畅，可能是 iptables 或者 server host 白名单造成的问题。
```bash
zabbix_get -s 10.10.1.1 -k system.uname
```

# zabbix proxy
如果 server cluster 规模不大，则我们可以采用被动模式（对 client 而言是被动的，被 zabbix server来拉取数据）。如果集群规模大，那么 zabbix server 的压力就很大了，如果要去抓取的客户端数量过于庞大。 还有一种建议用 proxy 模式的情况是跨地区监控，集中到一台中心 zabbix， 方便管理。
本次演示，我们用了三台机器，client: 10.66.236.98 , proxy: 10.66.236.99 , zabbix_server: 10.66.236.100 （可用上文的 docker 方式安装），非 docker 方式可用参考 [奥哥博客](https://wsgzao.github.io/post/zabbix/)

1. 首先是安装 zabbix proxy 在 10.66.236.99 
```bash
# zabbix proxy 的依赖就只有数据库了，用于存储配置信息 
yum install -y mariadb-server mariadb
# 启动 mariadb 服务 
systemctl start mariadb.service
systemctl enable mariadb.service
# 初始化 mysql 数据库，并配置 root 用户密码 
mysql_secure_installation
# 初始化，然后输入密码

# 创建 zabbix_proxy 数据库 
mysql -uroot -p

mysql> create database zabbix_proxy character set utf8 collate utf8_bin;
mysql> grant all privileges on zabbix_proxy.* to zabbix@localhost identified by 'zabbix';
mysql> quit;

# 部署 zabbix_proxy 4.2 版本的
rpm -Uvh  https://repo.zabbix.com/zabbix/4.2/rhel/7/x86_64/zabbix-release-4.2-2.el7.noarch.rpm
yum install -y zabbix-proxy-mysql
zcat /usr/share/doc/zabbix-proxy-mysql*/schema.sql.gz | mysql -uzabbix -p zabbix_proxy
#输入密码: zabbix

# 配置数据库用户及密码 
vim /etc/zabbix/zabbix_proxy.conf
Server=10.66.236.100   #填写 zabbix server 的IP
Hostname=zabbix_proxy
DBName=zabbix_proxy
DBUser=zabbix
DBPassword=zabbix

# 网页上配置 Zabbix Server Proxy
Administration -> Proxies -> Create proxy
Proxy name: zabbix_proxy
Proxy mode: Active
Proxy address: 10.66.236.99


# 启动 zabbix_proxy
systemctl restart zabbix-proxy
```

如果是 Active 模式，这里不需要填写 IP， Passive 被动模式才一定要填写 IP 
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20190822173033.png)

2. 修改 client 端的配置 
```bash
# 修改 zabbix_agent 配置指向 zabbix_proxy
vim /etc/zabbix/zabbix_agentd.conf
ServerActive=10.66.236.99  #proxy 的 IP
Hostname=10.66.236.98  #本机的 IP

# 重启 zabbix_agent
systemctl restart zabbix-agent
```

3. 在网页上添加 host

![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20190822173036.png)  
一定要选择 proxy，IP 就填写 0.0.0.0  还有一个非常重要的就是 template 所有的 item 需要采用 Zabbix agent (active)




# 遇到的坑
1. 在测试 proxy 时，host name 不匹配造成的，我最开始随便填了一个 hostname ， 然后发现和 agent config 不一致，又在zabbix 网页上更新为 10.66.236.98 ，感觉数据库并没有更新，所以导致 host name 匹配不了，一直报错 no active checks on server no active checks on server [10.66.236.99:10051]: host [10.66.236.98] not found   
解决的办法是：重新删除了 host , 再添加上 ，一次性填对了 host name 10.66.236.98 就可以了
