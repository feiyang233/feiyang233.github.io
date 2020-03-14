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