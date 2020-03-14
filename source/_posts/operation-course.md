---
title: Operation Course Record
date: 2020-02-13 18:56:35
tags: 
  - shell
  - nginx
categories: Linux
---
This blog is for record what I learn from this [Operation Course](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=42#/detail/pc?id=1546)

<!--more-->
# shell
## SHORTCUTS
Ctrl + r：find the histroy
Ctrl + l：clear the terminal
Ctrl + a \ Ctrl + e：move to head\end in row
Ctrl + w \ Ctrl + k：delete before\after cursor

ZZ : VIM save and quit

Ctrl+C: Interrupt (kill) the current foreground process running in in the terminal. This sends the SIGINT signal to the process, which is technically just a request—most processes will honor it, but some may ignore it.
Ctrl+Z: Suspend the current foreground process running in bash. This sends the SIGTSTP signal to the process. To return the process to the foreground later, use the fg process_name command.
Ctrl+D: Close the bash shell. This sends an EOF (End-of-file) marker to bash, and bash exits when it receives this marker. This is similar to running the exit command.

top SHORTCUTS  

Shift + p：Sort by CPU
Shift + m：Sort by Memory

## find large file
```shell
du -x --max-depth=1 / |sort -k1 -nr
du -x -d 1 -h / |sort -k1 -nr


# find which folder has most inode
find -type f | awk -F / -v OFS=/ '{$NF="";dir[$0]++}END{for(i in dir)print dir[i]" "i}'| sort -k1 -nr | head

769 ./wrk/obj/openssl-1.1.1b/test/
551 ./.cache/mozilla/firefox/hha91z74.default-release/cache2/entries/
462 ./wrk/obj/openssl-1.1.1b/doc/man3/
229 ./wrk/obj/LuaJIT-2.1.0-beta3/src/
212 ./wrk/obj/openssl-1.1.1b/test/certs/
207 ./wrk/obj/openssl-1.1.1b/apps/
199 ./wrk/obj/openssl-1.1.1b/crypto/asn1/
194 ./.cache/gnome-software/icons/
185 ./wrk/obj/openssl-1.1.1b/crypto/evp/
157 ./wrk/obj/openssl-1.1.1b/test/recipes/
```
## rename
```shell

# find consumer.xml file and replace all aaaaaa to bbbbbb
find ./ -type f -name consumer.xml -exec sed -i "s/aaaaaa/bbbbbb/g" {} \;

# find all .txt file and compress then copy to /home/.
(find . -name "*.txt"|xargs tar -cvf test.tar) && cp -f test.tar /home/.
```

## network
```shell
# network status
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'

# get IP 
ip a|grep "global"|awk '{print $2}'| awk -F/ '{print $1}'
```
# nginx
1. [Nginx是什么？能干什么？](https://mp.weixin.qq.com/s/IRQq3h4mur60rjVfcLsBzA)

## cpu affinity
```nginx
worker_cpu_affinity auto
```
[offical document](http://nginx.org/en/docs/ngx_core_module.html#worker_cpu_affinity)

## epoll
Add in kernel 2.6
1. epoll's event flow model is thread-safe;
2. epoll follow the selection model and improve efficiency;
3. epoll is event-driven. You can select the relevant state of the entire file to be scanned according to your needs. epoll avoids replacing the scanned file according to the event. You can directly call the callback function, which is more efficient.
4. Removed the maximum limit on the number of file sizes that can be monitored by the temporary process inside the select model (1024). If you have used an earlier version of Apache, the selection model it uses will be delayed when the request exceeds 1000. The request is wrong, and the performance will be significantly improved by using Nginx.
5. In addition, an optimization place involved in the events {} configuration is worker_connections, which is also set in events. Through the above learning, we know the role of the worker thread, and the connections supported by each worker thread are Limited, it will be set to 1024 by default here, and when we are dealing with high concurrency scenarios, it is often too low to split worker threads to 1024. It is recommended that you increase the worker_connections, you can refer to the actual business needs Nginx processing Maximum to increase this setting.

## sendfile
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20200216173928.png)
refer [docs](http://nginx.org/en/docs/http/ngx_http_core_module.html#sendfile)
```nginx
location /video/ {
    sendfile       on;
    tcp_nopush     on;
    aio            on;
}
```

## gzip
[docs](http://nginx.org/en/docs/http/ngx_http_gzip_module.html)
1. `gzip on` is responsible for turning on the compression function of the backend;
2. `gzip_buffer 16 8k` means to set the memory space of Nginx when processing file compression;
3. `gzip_comp_level 6` indicates the compression level of Nginx when processing compression. Generally, the higher the level, the larger the compression ratio. However, it does not mean that the larger the compression ratio, the better. It is still necessary to choose a suitable compression ratio according to the actual situation. The compression ratio is too large. Affects performance. Compression ratio is too small to achieve the desired effect. Generally, it is recommended that you set it to 6 to be more appropriate.
4. `gzip_http_version 1.1` means only compress the HTTP 1.1 version of the protocol;
5. `gzip_min_length 256` means that compression is performed only when the length is greater than the minimum 256 bytes, and if it is less than this length, no compression is performed;
6. `gzip_proxied any` means that Nginx as a reverse proxy sets some gzip compression policies based on the information returned by the backend server;
7. `gzip_vary on` indicates whether to send Vary: Accept_Encoding response header field, to realize that the receiver server is gzip compressed;
8. `application / vnd.ms-fontobject image / x-icon`; gip compression type;
9. g`zip_disable "msie6"`; Turn off compression for IE6.

## Timing Variables
+ [request_time](https://nginx.org/en/docs/http/ngx_http_core_module.html?&_ga=2.258153418.1195201124.1581934594-512055108.1581845805#var_request_time) – Full request time, starting when NGINX reads the first byte from the client and ending when NGINX sends the last byte of the response body
+ [upstream_connect_time](https://nginx.org/en/docs/http/ngx_http_upstream_module.html?&_ga=2.258153418.1195201124.1581934594-512055108.1581845805#var_upstream_connect_time) – Time spent establishing a connection with an upstream server
+ [upstream_header_time](https://nginx.org/en/docs/http/ngx_http_upstream_module.html?&_ga=2.57291882.1195201124.1581934594-512055108.1581845805#var_upstream_header_time) – Time between establishing a connection to an upstream server and receiving the first byte of the response header
+ [upstream_response_time](https://nginx.org/en/docs/http/ngx_http_upstream_module.html?&_ga=2.258153418.1195201124.1581934594-512055108.1581845805#var_upstream_response_time) – Time between establishing a connection to an upstream server and receiving the last byte of the response body

## nginx cache
1. ssl_session_cache
2. open_file_cache
```nginx
location / {
    index index.htm index.html;
    expires max;  
    open_file_cache max=1000 inactive=20s; 
    open_file_cache_valid 30s; 
    open_file_cache_min_uses 2; 
    open_file_cache_errors on;
}
```

3. proxy_cache_path
```nginx
proxy_cache_path /path/to/cache levels=1:2 keys_zone=my_cache:10m max_size=10g inactive=60mlocation / {
        proxy_cache my_cache;
        …
}
```

## load balance
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20200229125517.png)
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20200229125522.png)
nginx LB general config like follow
```nginx
http {
        …
        upstream app_servers {
                 server ip1:port1;
                 server ip2:port2;
                 server ip3:port3;
        }
        server {
         …
              location / {
                      proxy_pass http://app_servers; 
              }
        ….
        }
}
```
1. real ip
```nginx
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Real-IP $remote_addr;
```

2. Host forward
```nginx
proxy_set_header Host $host ;
proxy set_header Host www.vipumi.com; 
```

3. session miss
```nginx
# ip_hash
http {
        …
        upstream app_servers {
                 ip_hash;
                 server ip1:port1;
                 server ip2:port2;
                 server ip3:port3;
        }
        server {
         …
              location / {
                  proxy_pass http://app_servers; 
              }
        ….
        }
}


# URL_hash
http {
        …
        upstream app_servers {
            hash $request_uri;
                 server ip1:port1;
                 server ip2:port2;
                 server ip3:port3;
        }
        server {
         …
              location / {
                  proxy_pass http://app_servers;   
              }
        ….
        }
}

# session copy between upstream servers

# session share: store in zookeeper 
```
4. live ping
```nginx
# use taobao Tengine module

check interval=3000 rise=2 fall=5 timeout=1000 type=http;  //定义检查间隔、周期、时间
check_keepalive_requests 100;  //一个连接发送的请求数
check_http_send “HEAD / HTTP/1.1\r\nConnection: keep-alive\r\n\r\n”; //定义健康检查方式
check_http_expect_alive http_2xx http_3xx;  //判断后端返回状态码
```

# dynamic upstream
![](https://raw.githubusercontent.com/fainyang/pictures/master/img/20200301123058.png)

## nginx config 
```nginx
worker_processes  1;


error_log  logs/error.log  info;

#pid        logs/nginx.pid;

events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    lua_shared_dict upstreams 10m;

    upstream local {
        server 127.0.0.1:81;
        server 127.0.0.1:82;
        server 127.0.0.1:83;
        balancer_by_lua_file "./ngx_lua/upstream.lua";
        #check interval=3000 rise=2 fall=5 timeout=1000;
    }

    server {
        listen       800;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            proxy_pass http://local;
        }

        location = /ups {
            default_type  text/plain;
            content_by_lua_file "./ngx_lua/upsops.lua";
        }

        location = /ups/check {
            default_type  text/plain;
            content_by_lua_file "./ngx_lua/srvcheck.lua";
        }
        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

    }
}
```
1. upstream.lua
```lua
local ups = ngx.shared.upstreams;
local upstream = require "ngx.upstream"
local get_servers = upstream.get_servers
local set_peer_down = upstream.set_peer_down

local curr_ups = upstream.current_upstream_name()
local srvs = get_servers(curr_ups)

-- 摘除主节点的服务节点，传入的addr对应于主节点的name属性
function set_primary_server_down(addr)
    local peers = upstream.get_primary_peers(curr_ups)
    for i = 1, #peers do
        local peer = peers[i]
        if peer.name == addr then
            local ok, err = set_peer_down(curr_ups,false,peer.id,true)
            if not ok then
                ngx.log(ngx.ERR,curr_ups.." failed to set peer down: "..err)
                return false
            end
            return true
        end
    end
    return false
end

-- 摘除备份节点的服务节点，传入的addr对应于备份节点的name属性
function set_backup_server_down(addr)
    local peers = upstream.get_backup_peers(curr_ups)
    for i = 1, #peers do
        local peer = peers[i]
        if peer.name == addr then
            local ok, err = set_peer_down(curr_ups,true,peer.id,true)
            if not ok then
                ngx.log(ngx.ERR,curr_ups.." failed to set peer down: "..err)
                return false
            end
            return true
        end
    end
    return false
end

-- 恢复主节点的服务节点，传入的addr对应于主节点的name属性
function set_primary_server_up(addr)
    local peers = upstream.get_primary_peers(curr_ups)
    for i = 1, #peers do
        local peer = peers[i]
        if peer.name == addr then
            local ok, err = set_peer_down(curr_ups,false,peer.id,false)
            if not ok then
                ngx.log(ngx.ERR,curr_ups.." failed to set peer down: "..err)
                return false
            end
            return true
        end
    end
    return false
end

-- 恢复备份节点的服务节点，传入的addr对应于备份节点的name属性
function set_backup_server_up(addr)
    local peers = upstream.get_backup_peers(curr_ups)
    for i = 1, #peers do
        local peer = peers[i]
        if peer.name == addr then
            local ok, err = set_peer_down(curr_ups,true,peer.id,false)
            if not ok then
                ngx.log(ngx.ERR,curr_ups.." failed to set peer down: "..err)
                return false
            end
            return true
        end
    end
    return false
end

for i,peer in ipairs(srvs) do
    local isdown = ups:get(peer["name"])
    -- 摘除服务节点
    if isdown == 1 then
        if peer.backup == nil then
            -- peer.addr: socket 地址，可能是Lua字符串或lua字符串的数组.
            local addr = peer.addr
            if type(addr) == "table" then
                for _,v in ipairs(addr) do
                    if set_primary_server_down(v) == true then
                        ngx.log(ngx.INFO,curr_ups.." "..peer["name"].." set down success.")
                    else
                        ngx.log(ngx.ERR,curr_ups.." "..peer["name"].." set down failed.")
                    end
                end
            else
                if set_primary_server_down(addr) == true then
                    ngx.log(ngx.INFO,curr_ups.." "..peer["name"].." set down success.")
                else
                    ngx.log(ngx.ERR,curr_ups.." "..peer["name"].." set down failed.")
                end
            end
        elseif peer.backup == true then
            local addr = peer.addr
            if type(addr) == "table" then
                for _,v in ipairs(addr) do
                    if set_backup_server_down(v) == true then
                        ngx.log(ngx.INFO,curr_ups.." "..peer["name"].." set down success.")
                    else
                        ngx.log(ngx.ERR,curr_ups.." "..peer["name"].." set down failed.")
                    end
                end
            else
                if set_backup_server_down(addr) == true then
                    ngx.log(ngx.INFO,curr_ups.." "..peer["name"].." set down success.")
                else
                    ngx.log(ngx.ERR,curr_ups.." "..peer["name"].." set down failed.")
                end
            end
        end
    -- 恢复服务节点
    elseif isdown == 0 then
        if peer.backup == nil then
            local addr = peer.addr
            if type(addr) == "table" then
                for _,v in ipairs(addr) do
                    if set_primary_server_up(v) == true then
                        ups:delete(peer["name"])
                        ngx.log(ngx.INFO,curr_ups.." "..peer["name"].." set up success.")
                    else
                        ngx.log(ngx.ERR,curr_ups.." "..peer["name"].." set up failed.")
                    end
                end
            else
                if set_primary_server_up(addr) == true then
                    ups:delete(peer["name"])
                    ngx.log(ngx.INFO,curr_ups.." "..peer["name"].." set up success.")
                else
                    ngx.log(ngx.ERR,curr_ups.." "..peer["name"].." set up failed.")
                end
            end
        elseif peer.backup == true then
            local addr = peer.addr
            if type(addr) == "table" then
                for _,v in ipairs(addr) do
                    if set_backup_server_up(v) == true then
                        ups:delete(peer["name"])
                        ngx.log(ngx.INFO,curr_ups.." "..peer["name"].." set up success.")
                    else
                        ngx.log(ngx.ERR,curr_ups.." "..peer["name"].." set up failed.")
                    end
                end
            else
                if set_backup_server_up(addr) == true then
                    ups:delete(peer["name"])
                    ngx.log(ngx.INFO,curr_ups.." "..peer["name"].." set up success.")
                else
                    ngx.log(ngx.ERR,curr_ups.." "..peer["name"].." set up failed.")
                end
            end
        end
    end
end
```

2. upsops.lua
```lua
local cjson = require "cjson";
local upstream = require "ngx.upstream"
local ups = ngx.shared.upstreams;

local get_upstreams = upstream.get_upstreams
local all_ups = upstream.get_upstreams()

local op = ngx.req.get_uri_args()["op"];
local server = ngx.req.get_uri_args()["server"];

if op == nil or server == nil then
    ngx.say("usage: /ups?op=add&server=192.168.56.101:8080");
    return
end

function down_server(upstream_name,server)
    local ret = false
    local perrs = upstream.get_servers(upstream_name)
    local avail = #perrs

    for i = 1, #perrs do
        local peer = perrs[i]
        local isdown = ups:get(peer["name"])
        if isdown == 1 then
            avail = avail - 1
            ngx.log(ngx.ERR,"## peer.down: true  ups: "..upstream_name.." peer.name: "..peer.name.." server: "..server.." avail: "..avail)
        end
        if peer.name == server then
            ret = true
        end
    end
    if not ret then
        avail = 2
    end
    return avail
end

if op == "add" then
    ups:set(server,0)
    ngx.say(cjson.encode({code="A00001", msg="add a server success.",data={server}}))
elseif op == "del" then
    local isdown = ups:get(server)
    if isdown == 1 then
        ngx.say(cjson.encode({code="A00001", msg="del a server success.",data={server}}))
        return
    end
    for _,u in ipairs(all_ups) do
        local srvs = upstream.get_servers(u)
        for i,peer in ipairs(srvs) do
            local down_svc = down_server(u,server)
            ngx.log(ngx.ERR,"## down_svc: "..down_svc.." upstream: "..u.." server: "..server)
            if down_svc < 2 then
                ngx.log(ngx.ERR,u.." You cat not set peer down: "..peer["name"])
                ngx.say(cjson.encode({code="E00001", msg="You cat not set peer down: "..peer["name"],data={peer["name"]}}))
                return
            end
        end
    end

    ups:set(server,1)
    ngx.say(cjson.encode({code="A00001", msg="del a server success.",data={server}}))
    return
else
    ngx.say(cjson.encode({code="E00001", msg="do none.",data={}}))
    return
end
```

3. srvcheck.lua
```lua
local cjson = require "cjson";
local ups = ngx.shared.upstreams;
local servers = {}
for _,v in pairs(ups:get_keys(0)) do
   servers[v] = ups:get(v)
end
ngx.say(cjson.encode({code="A00001", msg="OK", data=servers}))
```

# curl
- 21 curl exercises https://jvns.ca/blog/2019/08/27/curl-exercises/
- answer http://blog.lujun9972.win/blog/2019/09/11/curl%E7%BB%83%E4%B9%A0/
- https://wsgzao.github.io/post/curl/
```bash
# 语法 
curl [option] [url]

# 最简单的使用，获取服务器内容，默认将输出打印到标准输出中(STDOUT) 中。
curl http://www.centos.org

# 添加 - v 参数可以看到详细解析过程，通常用于 debug
curl -v http://www.centos.org

# curl 发送 Get 请求
curl URL
curl URL -O 文件绝对路径
 
# curl 发送 post 请求

# 请求主体用 json 格式
curl -X POST -H 'content-type: application/json' -d @json 文件绝对路径 URL
curl -X POST -H 'content-type: application/json' -d 'json 内容' URL
# 请求主体用 xml 格式
curl -X POST -H 'content-type: application/xml' -d @xml 文件绝对路径 URL
curl -X POST -H 'content-type: application/xml' -d 'xml 内容' URL
 
# 设置 cookies
curl URL --cookie "cookie 内容"
curl URL --cookie-jar cookie 文件绝对路径
 
# 设置代理字符串
curl URL --user-agent "代理内容"
curl URL -A "代理内容"
 
# curl 限制带宽
curl URL --limit-rate 速度
 
# curl 认证
curl -u user:pwd URL
curl -u user URL
 
# 只打印 http 头部信息
curl -I URL
curl -head URL

# 末尾参数
--progress  显示进度条
--silent    不现实进度条

# 不需要修改 / etc/hosts，curl 直接解析 ip 请求域名
# 将 http://example.com 或 https://example.com 请求指定域名解析的 IP 为 127.0.0.1
curl --resolve example.com:80:127.0.0.1 http://example.com/
curl --resolve example.com:443:127.0.0.1 https://example.com/

-X/--request [GET|POST|PUT|DELETE|…]  指定请求的 HTTP 方法
-H/--header                           指定请求的 HTTP Header
-d/--data                             指定请求的 HTTP 消息体（Body）
-v/--verbose                          输出详细的返回信息
-u/--user                             指定账号、密码
-b/--cookie                           读取 cookie  

# 典型的测试命令为：
curl -v -X POST -H "Content-Type: application/json" http://127.0.0.1:8080/user -d'{"username":"admin","password":"admin1234"}'...

# 测试 get 请求
curl http://www.linuxidc.com/login.cgi?user=test001&password=123456

# 测试 post 请求
curl -d "user=nickwolfe&password=12345" http://www.linuxidc.com/login.cgi

# 请求主体用 json 格式
curl -X POST -H 'content-type: application/json' -d @json 文件绝对路径 URL
curl -X POST -H 'content-type: application/json' -d 'json 内容' URL
 
# 请求主体用 xml 格式
curl -X POST -H 'content-type: application/xml' -d @xml 文件绝对路径 URL
curl -X POST -H 'content-type: application/xml' -d 'xml 内容' URL

# 发送 post 请求时需要使用 - X 选项，除了使用 POST 外，还可以使用 http 规范定义的其它请求谓词，如 PUT,DELETE 等
curl -XPOST url

# 发送 post 请求时，通常需要指定请求体数据。可以使用 - d 或 --data 来指定发送的请求体。
curl -XPOST -d "name=leo&age=12" url

# 如果需要对请求数据进行 urlencode, 可以使用下面的方式：
curl -XPOST --data-urlencode "name=leo&age=12" url

# 此外发送 post 请求还可以有如下几种子选项：
–data-raw
–data-ascii
–data-binary

# To retrieve the job config.xml
curl -X GET '<jenkinshost>/job/<jobname>/config.xml' -u username:API_TOKEN -o <jobname>.xml

# to use this config to create a new job
curl -s -XPOST '<jenkinshost>/createItem?name=<jobname>' -u username:API_TOKEN --data-binary @<jobname>.xml -H "Content-Type:text/xml"

# get all jenkins jobs
curl -X GET '<jenkinshost>/api/json?pretty=true' -u username:API_TOKEN -o jobs.json

# get jenkins view
curl -X GET '<jenkinshost>/view/<viewname>/api/json' -u username:API_TOKEN -o view.json
```

# unixbench & fio
Install unixbench can refer this https://www.ostechnix.com/unixbench-benchmark-suite-unix-like-systems/

```bash
# for ubuntu

sudo apt-get install libx11-dev libgl1-mesa-dev libxext-dev perl perl-modules make git

git clone https://github.com/kdlucas/byte-unixbench.git

cd byte-unixbench/UnixBench/

./Run

# output 
root@ubuntu:/tmp/byte-unixbench/UnixBench# ./Run 
make all
make[1]: Entering directory '/tmp/byte-unixbench/UnixBench'
make distr
make[2]: Entering directory '/tmp/byte-unixbench/UnixBench'
Checking distribution of files
./pgms  exists
./src  exists
./testdir  exists
./tmp  exists
./results  exists
make[2]: Leaving directory '/tmp/byte-unixbench/UnixBench'
make programs
make[2]: Entering directory '/tmp/byte-unixbench/UnixBench'
make[2]: Nothing to be done for 'programs'.
make[2]: Leaving directory '/tmp/byte-unixbench/UnixBench'
make[1]: Leaving directory '/tmp/byte-unixbench/UnixBench'
sh: 1: 3dinfo: not found

   #    #  #    #  #  #    #          #####   ######  #    #   ####   #    #
   #    #  ##   #  #   #  #           #    #  #       ##   #  #    #  #    #
   #    #  # #  #  #    ##            #####   #####   # #  #  #       ######
   #    #  #  # #  #    ##            #    #  #       #  # #  #       #    #
   #    #  #   ##  #   #  #           #    #  #       #   ##  #    #  #    #
    ####   #    #  #  #    #          #####   ######  #    #   ####   #    #

   Version 5.1.3                      Based on the Byte Magazine Unix Benchmark

   Multi-CPU version                  Version 5 revisions by Ian Smith,
                                      Sunnyvale, CA, USA
   January 13, 2011                   johantheghost at yahoo period com

------------------------------------------------------------------------------
   Use directories for:
      * File I/O tests (named fs***) = /tmp/byte-unixbench/UnixBench/tmp
      * Results                      = /tmp/byte-unixbench/UnixBench/results
------------------------------------------------------------------------------
```


Install fio https://github.com/axboe/fio
```bash
sudo apt update
sudo apt install fio

# run write test
feiyang@ubuntu:~$  fio --name=randwrite --ioengine=libaio --iodepth=1 --rw=randwrite --bs=4k --direct=0 --size=512M --numjobs=2 --runtime=240 --group_reporting

randwrite: (g=0): rw=randwrite, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=1
...
fio-3.1
Starting 2 processes
randwrite: Laying out IO file (1 file / 512MiB)
randwrite: Laying out IO file (1 file / 512MiB)
Jobs: 2 (f=2)
randwrite: (groupid=0, jobs=2): err= 0: pid=1910: Sat Mar 14 12:48:59 2020
  write: IOPS=204k, BW=799MiB/s (838MB/s)(1024MiB/1282msec)
    slat (nsec): min=1929, max=909527, avg=7784.12, stdev=14736.01
    clat (nsec): min=348, max=1920.9k, avg=613.38, stdev=5441.37
     lat (usec): min=2, max=1925, avg= 8.50, stdev=15.99
    clat percentiles (nsec):
     |  1.00th=[   378],  5.00th=[   386], 10.00th=[   390], 20.00th=[   394],
     | 30.00th=[   398], 40.00th=[   402], 50.00th=[   410], 60.00th=[   418],
     | 70.00th=[   430], 80.00th=[   852], 90.00th=[   964], 95.00th=[   988],
     | 99.00th=[  1336], 99.50th=[  1864], 99.90th=[ 22144], 99.95th=[ 31872],
     | 99.99th=[209920]
   bw (  KiB/s): min=432536, max=539248, per=57.01%, avg=466269.00, stdev=49219.22, samples=4
   iops        : min=108134, max=134812, avg=116567.00, stdev=12304.94, samples=4
  lat (nsec)   : 500=78.66%, 750=1.10%, 1000=16.26%
  lat (usec)   : 2=3.64%, 4=0.11%, 10=0.10%, 20=0.03%, 50=0.08%
  lat (usec)   : 100=0.01%, 250=0.01%, 500=0.01%, 750=0.01%
  lat (msec)   : 2=0.01%
  cpu          : usr=1.11%, sys=98.68%, ctx=24, majf=0, minf=18
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwt: total=0,262144,0, short=0,0,0, dropped=0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
  WRITE: bw=799MiB/s (838MB/s), 799MiB/s-799MiB/s (838MB/s-838MB/s), io=1024MiB (1074MB), run=1282-1282msec

Disk stats (read/write):
  sda: ios=0/3020, merge=0/2481, ticks=0/450, in_queue=4, util=30.57%

# run read test
feiyang@ubuntu:~$  fio --name=randread --ioengine=libaio --iodepth=16 --rw=randread --bs=4k --direct=0 --size=512M --numjobs=4 --runtime=240 --group_reporting
randread: (g=0): rw=randread, bs=(R) 4096B-4096B, (W) 4096B-4096B, (T) 4096B-4096B, ioengine=libaio, iodepth=16
...
fio-3.1
Starting 4 processes
randread: Laying out IO file (1 file / 512MiB)
randread: Laying out IO file (1 file / 512MiB)
randread: Laying out IO file (1 file / 512MiB)
randread: Laying out IO file (1 file / 512MiB)
Jobs: 4 (f=4): [r(4)][100.0%][r=81.3MiB/s,w=0KiB/s][r=20.8k,w=0 IOPS][eta 00m:00s]
randread: (groupid=0, jobs=4): err= 0: pid=1919: Sat Mar 14 12:51:02 2020
   read: IOPS=20.4k, BW=79.7MiB/s (83.6MB/s)(2048MiB/25686msec)
    slat (usec): min=65, max=33169, avg=192.80, stdev=110.46
    clat (usec): min=2, max=48551, avg=2939.25, stdev=573.24
     lat (usec): min=92, max=49600, avg=3132.50, stdev=601.03
    clat percentiles (usec):
     |  1.00th=[ 2606],  5.00th=[ 2671], 10.00th=[ 2704], 20.00th=[ 2737],
     | 30.00th=[ 2769], 40.00th=[ 2835], 50.00th=[ 2868], 60.00th=[ 2933],
     | 70.00th=[ 2999], 80.00th=[ 3097], 90.00th=[ 3228], 95.00th=[ 3359],
     | 99.00th=[ 3621], 99.50th=[ 3752], 99.90th=[ 5342], 99.95th=[ 9503],
     | 99.99th=[36963]
   bw (  KiB/s): min=17683, max=22104, per=25.00%, avg=20407.93, stdev=962.56, samples=204
   iops        : min= 4420, max= 5526, avg=5101.91, stdev=240.65, samples=204
  lat (usec)   : 4=0.01%, 100=0.01%, 250=0.01%, 500=0.01%, 750=0.01%
  lat (usec)   : 1000=0.01%
  lat (msec)   : 2=0.03%, 4=99.74%, 10=0.18%, 20=0.03%, 50=0.02%
  cpu          : usr=0.04%, sys=63.18%, ctx=452451, majf=0, minf=92
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=100.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.1%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwt: total=524288,0,0, short=0,0,0, dropped=0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=16

Run status group 0 (all jobs):
   READ: bw=79.7MiB/s (83.6MB/s), 79.7MiB/s-79.7MiB/s (83.6MB/s-83.6MB/s), io=2048MiB (2147MB), run=25686-25686msec

Disk stats (read/write):
  sda: ios=523624/5, merge=0/5, ticks=84769/1, in_queue=88, util=99.53%
```


