## 1、应用场景
> 部署一个proxy做反向代理，根据Http请求中的机房的名字，将请求转发到不同机房的后端服务器上。
## 2、OpenResty

官网上是这么解释的： OpenResty effectively turns the nginx server into a powerful web app server, in which the web developers can use the Lua programming language to script various existing nginx C modules and Lua modules and construct extremely high-performance web applications that are capable to handle 10K ~ 1000K+ connections in a single box

我自己的理解是：OpenResty包含两个东西，Nginx和LuaJIT。借助于Nignx可以进行非阻塞的方式处理请求，然后通过Lua的脚本对接收到的请求进行处理，比如确定请求发往的目的服务器。

现在开始我们的实践，首先描述下实验中Http请求的头部格式。然后展示下nginx的配置、lua脚本、upstream的配置。


## 3、请求头部

这里的Zone是必备参数，

url： XX.api.ucloud.cn
Request：
{
  "Action":"GetNetTC",
  "EIP":"120.132.6.X",
  "Zone": "cn-sh2-01", #必备参数，需要根据Zone将请求转发到对应机房的服务器上
  "Backend": "UNetTool",
}

## 4、文件目录

conf: nignx和unet-tool服务器的配置文件
lua-script: lua的两个脚本文件
fabfile.py: 自动部署的python脚本


## 5、Nignx配置

Nignx配置文件中除了指定监听端口、日志位置等，还需要配置proxy_pass，其值是由lua-script/main.lua确定的。

```nginx

worker_processes 4;
pid logs/nginx.pid;

error_log logs/error.log info;

events {

    worker_connections 2048;

}

http {

    log_format  upstream  '$time_iso8601 $remote_addr $host $request_time $upstream_response_time $request $request_body $status $upstream_addr';

    # lua settings
    lua_package_path 'lua-script/?.lua;;';

    # region upstream
    include upstreams.conf;    #引入各机房服务器的配置

    server {
        listen 0.0.0.0:2003; #部署脚本中会将0.0.0.0替换成proxy的Ip地址。
        access_log  logs/unettool-proxy-access.log upstream;

        error_log   logs/unettool-proxy-error.log;



        location / {

            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

            proxy_set_header Host $http_host;

            proxy_set_header X-Real-IP $remote_addr;

            proxy_set_header Connection "";

            proxy_http_version 1.1;



            set $backend ''; #定义一个变量backend

            rewrite_by_lua_file lua-script/main.lua; #更新变量backend


            proxy_pass http://$backend; #将请求发往backend

            error_page 500 502 = @backend_error;

        }



        location @backend_error {

            content_by_lua_file lua-script/backend_error.lua;

        }

    }

} 

```

## 6、Lua脚本

mian.lua是用途是确定某个请求发往哪个后端的，将结果赋值给backend。部分代码如下所示
最重要的代码其实只有两行，根据请求头部的zone，得到_upstream的名字，然后赋值给变量backend
    _upstream = 'unettool-proxy-'..tostring(zone)
    ngx.var.backend = _upstream
```lua

function _send_error_response(err)
    -- 一些设置
    ngx.say('{"RetCode": 230, "Message": "Param [Zone] is required"}')
end

function _get_req_params()
    local method_name = ngx.req.get_method()
    local args = {}
    local err = nil

    if method_name == 'GET' then
        args = ngx.req.get_uri_args()
    elseif method_name == 'POST' then
        ngx.req.read_body()
        local content_type = ngx.req.get_headers()['Content-Type']
        if content_type == 'application/x-www-form-urlencoded' then
            args, err = ngx.req.get_post_args()
        elseif content_type == 'application/json' then
            args, err = ngx.req.get_body_data() 
            if args ~= ngx.null then
                args = cjson.decode(args)
            end
        end
    end

    return args
end

local args = _get_req_params()
if args == nil then
    _send_error_response('fetch request params error')
    return
end

local action = args['Action']

local zone = args['Zone']
if zone == nil or zone == '' then
    _send_error_response('missing Zone')
    return
end

_upstream = 'unettool-proxy-'..tostring(zone)
if _upstream == nil or _upstream == '' then
    _send_error_response('Zone or SrcZone is illegal')
    return 
end

ngx.var.backend = _upstream

```

## 7、Upstream的配置

示例如下：

```nginx

upstream unettool-proxy-cn-bj1-01{
    keepalive 1000;
    server 172.X.X.X:2003;
}
upstream unettool-proxy-cn-sh2-01{
    keepalive 1000;
    server 172.X.X.X:2003;
}
upstream unettool-proxy-cn-sh2-02{
    keepalive 1000;
    server 172.X.X.X:2003;
}
```

到这里基本上，就算完成了，为了方便我们快速部署，写了一个简单的fabfile文件。

## 8、fabfile自动部署

这一步没什么可说的，就是用python的fabric库执行一些远程和本地的命令。

```python 

#! /usr/bin/env python

from fabric.api import local,env,run,local,settings,put

env.hosts=['172.X.X.X', '172.X.X.X']

def deploy(branch='from_dezho',tag='v0.1'):
    code_dir = '/data/unet-tool'
    local_tmp_code_dir = '~/tmp/unet-tool'
    put('openresty-1.9.15.1.tar.gz', '/tmp')
    run('cd /tmp/ && tar xvf openresty-1.9.15.1.tar.gz')
    run('yum install pcre')
    run('yum install readline-devel pcre-devel openssl-devel gcc')
    run('cd /tmp/openresty-1.9.15.1 && ./configure && make && make install')
    local('tar cf unettool-proxy.tar ../unettool-proxy --exclude openresty-1.9.15.1.tar.gz')
    put('./unettool-proxy.tar', '/data/')
    run('cd /data/unettool-proxy/ && /usr/local/openresty/nginx/sbin/nginx -p /data/unettool-proxy/ -c conf/nginx.conf -s stop')
    run('rm -rf /data/unettool-proxy')
    run('cd /data && tar xf unettool-proxy.tar')
    run('rm /data/unettool-proxy.tar')
    run('sed -i "s/0.0.0.0/%s/g" /data/unettool-proxy/conf/nginx.conf ' % env.host )
    run('cd /data/unettool-proxy/ && /usr/local/openresty/nginx/sbin/nginx -p /data/unettool-proxy/ -c conf/nginx.conf')

def updateUpstreamConf():
    put('conf/upstreams.conf', '/data/unettool-proxy/conf/')
    run('cd /data/unettool-proxy/ && /usr/local/openresty/nginx/sbin/nginx -p /data/unettool-proxy/ -c conf/nginx.conf -s stop')
    run('cd /data/unettool-proxy/ && /usr/local/openresty/nginx/sbin/nginx -p /data/unettoo
```
