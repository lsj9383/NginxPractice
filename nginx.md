# 一、Nginx概述
由于本人的Linux为CentOS, 因此相关命令和配置项仅限于CentOS。国内常用的Nginx及相关package:
* Nginx开源: http://nginx.org/
* Nginx商业版: https://www.nginx.com/
* openresty, 一个基于Nginx基础上进行了lua等相关模块的扩展
* tengine, 基于Nginx重构的一个框架，对Nginx本身的变更较大。

下载Nginx的安装包:
```sh
wget -c https://nginx.org/download/nginx-1.10.1.tar.gz
```

安装Nginx前通常需要安装相关依赖, 可以直接使用该命令进行安装:
```sh
yum -y install gcc g++ automake pcre-devel pcre zlib-devel zlib openssl-devel
```
* gcc/g++, c/c++的编译器, 用于编译ngx源码
* pcre/pcre-devel, ngx使用该库支持正则表达式解析
* zlib/zlib-devel, ngx响应进行gzip压缩的库
* openssl-devel, 用于支持HTTPS的库
* automake, 通常一些第三方模块可能会用到

## 1.1 构建的常用参数
* `--prefix=path`
* `--user=name`, 设置ngx的worker所属用户, 默认用户为nobody, 如果没有进行配置可能会导致无法访问等权限问题。也可以在ngx的配置文件中配置。
* `--group=name`, 设置ngx的worker所属组, 默认为非root的用户组。也可以在ngx的配置文件中配置。
* `--with-http_ssl_module`, 启用ssl, 默认未开启, 若不开启则无法使用https。
* `--with-http_v2_module`, 启用http 2.0, 默认未开启。

常用构建命令:
```sh
# 通用ngx
./configure --prefix=/home/user00/bin/openresty \
            --with-http_ssl_module \
            --add-module=${module_dir}

# openresty
./configure --prefix=/home/user00/bin/openresty \
            --with-http_ssl_module \
            --with-luajit \
            --without-http_redis2_module \
            --with-http_iconv_module \
            --with-http_postgres_module \
            --add-module=${module_dir}
```

## 1.2 Nginx安装包目录

# 二、内置变量
下面只列出的本人使用过的内置Nginx变量，其他内置变量并未列出，将会根据本人情况持续更新。更全面的内置变量可以看官方文档: [Alphabetical index of variables](http://nginx.org/en/docs/varindex.html)。

| var | desc |
| :------: | :------ |
| $request | 整个原始请求行 |
| $request_body | 请求的body, location中用到proxy_pass,fastcgi_pass,scgi_pass命令时该变量才有值，否则为空串 |
| $request_uri | 请求uri(不包含协议和域名，但是包含查询字符串), 不会受rewrite影响 |
| $request_method | 请求的方法，比如"GET"、"POST"等 |
| $uri | 请求uri(不包含协议、域名和查询字符串), 若经过了rewrite, 则$uri是rewrite后的值 |
| $args | 请求中的查询参数 |
| $query_string | 请求中的查询字符串, 等价于$args |
| $arg_XXXX | 请求中的参数XXXX对应的value |
| $is_args | 如果uri带有查询参数，返回"?"，否则返回空字符串 |
| $scheme | 所用的协议，比如http或者是https |
| $http_XXXX | 请求中headers的XXXX对应的value |
| $http_via | 最后一个访问服务器的Ip地址 |
| $http_x_forwarded_for | 相当于网络访问路径|
| $remote_addr | 客户端IP地址 |
| $remote_port | 客户端端口号 |
| $remote_user | 客户端用户名, 认证用, 由Basic authentication校验后提供 |

若有一个如下请求:
```sh
$ curl -v -H "Content-type: application/json" -d '{"name":"lsj"}' "http://github.com/lsj9383/NginxPractice?a=1&b=3"

# 请求体
> POST /lsj9383/NginxPractice HTTP/1.1
> User-Agent: curl/7.29.0
> Accept: */*
> Host: github.com
> Content-type: application/json
> Content-Length: 2
```

下面是一些可以确认的内置变量:
* $request为`POST /lsj9383/NginxPractice HTTP/1.1`
* $uri为`/lsj9383/NginxPractice`
* $request_uri为`/lsj9383/NginxPractice?a=1&b=2`
* $args为`a=1&b=2`
* $arg_a为`1`
* $is_args为`?`

# 三、常用配置
下面会列出每个语句块中的常用配置。更全面的配置可以看官方文档: [Alphabetical index of directives](http://nginx.org/en/docs/dirindex.html)

## 3.1 基础相关
```
Syntax: user user [group];
Default: user nobody nobody;
Context: main
```

## 3.2 性能优化相关
* `accept_mutex`，worker在accept前是否进行上锁的标识，默认为off。
    * 如果打开，nginx的worker会在accept前申请锁，新连接来的时候只会有一个worker被唤醒。
    * 如果关闭，新连接会让所有worker都被唤醒，即惊群。
    * 互联网高并发业务一般都是关闭，即允许惊群。一方面是 nginx 的 worker 不会很多，惊群影响不大，另一方面高并发有大量的连接进行建立，需要尽可能多的worker去获取连接，避免阻塞。
* `multi_accept`，目前默认是off。
    * 如果关闭，worker 一次只会 accept 一个 socket，
    * 如果打开，worker 一次会把 accept queue 中的 socket 全部取出。
    * 高并发的情况下，如果不能及时accept，会导致socket挤压，甚至被拒绝连接。高并发条件下需要及时清空accept queue。所以我们一般配置为on。

## 3.3 证书相关

## 3.4 限速相关
* `limit_req_zone key zone=name:size rate=rate [sync]`，key是指的对这个变量的出现速率进行限制；zone标识存储区域以及存储区域大小（其实就是漏桶的大小），超出漏洞的部分将会被拒绝；rate指的是具体速率。
* `limit_req zone=name [burst=number]` 关联到漏桶，受到漏桶流出速率的限制。burst用于处理突发流量。
* http://nginx.org/en/docs/http/ngx_http_limit_req_module.html

下面是一个对 URI 进行限速的 dmo:
```conf
http {
    limit_req_zone $uri zone=webapi:10m rate=500r/s;

    server {
        location ~ /webapi/.+ {
            limit_req zone=webapi burst=1000;
            proxy_pass http://upstream_server;
        }
    }
}
```

## 3.5 超时相关
* proxy_connect_timeout, Nginx 和 upsteam 的节点建立连接所用的超时时间，即当超过这个时间都没有建立好连接就放弃，直接返回客户端失败。
* proxy_send_timeout, Nginx 向 upstream 的节点两次成功写的超时时间，超时后 nginx 直接断开连接。
    * 一般业务后台压力过大，无法及时处理请求，也就没法及时从 socket 读取数据，这样 nginx 的写行为就会超时，断开链接。
    * 写等待时间超过了该时间，意味着两次成功读之间的时间一定超过该时间
* proxy_read_timeout, Nginx 从 upstream 的节点两次成功读的超时时间，超时后 nginx 直接断开连接。
    * nginx 将数据全部传给后台后，后台的处理耗时过长，nginx 的 socket 等待时间超过了该时间就会超时，断开连接。
    * 读等待时间超过了该时间，意味着两次成功读之间的时间一定超过该时间

## 3.6 缓存相关
Nginx 常用的缓存方式为 Nginx Proxy，其原理相对简单：
* 每次请求时都会计算请求的key，这个是一个 md5 值，根据不同的配置不同的请求计算的 key 方式可能不一样。
* 如果 key 之前未进行缓存，或者缓存过期，则请求逻辑后台，将响应进行缓存，写到磁盘中。
* 如果 key 之前已经被缓存，且未过期，则直接到磁盘 / 内存中获取响应。

缓存相关的命令如下：
* proxy_cache_path，指定缓存的目录、key的共享内存名称大小、缓存时间等。
```
Syntax: proxy_cache_path path [levels=levels] [use_temp_path=on|off] keys_zone=name:size [inactive=time] [max_size=size] [manager_files=number] [manager_sleep=time] [manager_threshold=time] [loader_files=number] [loader_sleep=time] [loader_threshold=time]
Default: —
Context: http
```
其中的参数如下：
    * path，指定缓存数据在磁盘中的落盘目录
    * levels=levels，数据在磁盘中的存储路径位 /${path}/${key}, key 是一个 MD5 值，levels 将 key 部分切分位多个部分，每个部分是一个目录。levels最多有3级，即最多有3个目录。
    * `use_temp_path=on|off`，所有的数据在落盘前都是存放在临时文件中的，如果`user_temp_path=on`则使用用户自定义的临时文件目录，由`proxy_temp_path`指定用户的临时文件目录。如果为off，则临时文件目录即缓存目录。
    * `keys_zone=name:size`，指定缓存的共享内存空间，共享内存空间在多进程之间共享被缓存的key，ngx会先在共享内存中判断请求的key是否存在，如果存在则到磁盘中获取数据。size是共享内存空间。* `inactive=time`，当 key 在 time 时间范围内都没有，该 key 所对应的文件将会从磁盘中移除。即便该 key 还未到过期时间。
    * `cache manager`，nginx将会启动一个 cache manager 进程，该进程的目的是定期检查缓存文件是否需要清理。
        * `max_size=size`，max_size参数设置的最大缓存大小。当超过此大小时，cache manager 进程将删除最近最少使用的数据。
        * `manager_files=number`，每次处理不超过number个文件。
        * `manager_threshold=time`，每次清理不能超过time耗时。
        * `manager_sleep=time`，两次清理之间的间隔。（上一次清理完毕到下一次开始清理）。
    * `cache loader`，nginx会定期启动要给cache loader，将磁盘中的缓存数据加载到内存中，避免频繁的磁盘读写操作。一次性加载所有内容会影响性能，因此ngx会控制每次加载到内存中的数据量。
        * `loader_files=number`，每次加载的文件数量。
        * `loader_threshold=time`，每次加载不超过time耗时。
        * `loader_sleep=time`，两次加载的时间间隔。

除去上述配置外，还有 purger 的配置，用于控制缓存文件的清理，这是商业项目支持的配置，这里暂不给出。
* proxy_cache，定义请求在该上下文中使用的缓存区域，该区域名和 proxy_cache_path 中的 keys_zone 对应。通过这个方式将请求的缓存和 proxy_cache_path 指定的缓存区域关联起来。
```
Syntax:	proxy_cache zone | off;
Default: proxy_cache off;
Context: http, server, location
```
*  proxy_cache_valid，设置了对于不同响应状态码的数据缓存时间。该指令必须要配置，否则无法成功缓存。如果code没有填写，则只对 200 301 302 的响应进行缓存。如果code为any则对所有响应进行缓存。
```
Syntax:	proxy_cache_valid [code ...] time;
Default:	—
Context:	http, server, location
```
* proxy_cache_key，定义请求的 key，当请求的 key  被缓存时，直接返回请求对应的内容。
```
Syntax:	proxy_cache_key string;
Default: proxy_cache_key $scheme$proxy_host$request_uri;
Context: http, server, location
```
* proxy_cache_methods，指定的会进行缓存的请求需要满足的HTTP方法。
```
Syntax:	proxy_cache_methods GET | HEAD | POST ...;
Default:	
proxy_cache_methods GET HEAD;
Context:	http, server, location
This directive appeared in version 0.7.59.
```
* proxy_cache_min_uses，当请求的次数达到该阈值，对响应进行缓存。
```
Syntax: proxy_cache_min_uses number;
Default: proxy_cache_min_uses 1;
Context: http, server, location
```
* proxy_no_cache，定义请求不被缓存的条件, 当指定的 string 不为空或者不为0时，响应将不被缓存。
```
Syntax:	proxy_no_cache string ...;
Default: —
Context: http, server, location
```
* proxy_cache_bypass，定义请求不读取缓存的条件。当指定的string不为空或者不为0时，响应不从缓存中获取，而是均直接请求后端。
```
Syntax:	proxy_cache_bypass string ...;
Default: —
Context: http, server, location
```
* proxy_cache_convert_head，是否允许ngx将head方法转换为get方法进行请求，以次将head请求进行缓存。如果禁用，则需要通过 proxy_cache_methods 来开启对 HEAD 的缓存。
```
Syntax:	proxy_cache_key string;
Default: proxy_cache_key $scheme$proxy_host$request_uri;
Context: http, server, location
```
* proxy_cache_use_stale，当后端服务器处于指定的状态时，认为服务器不可用，使用历史缓存数据来响应客户端。
```
Syntax:	proxy_cache_use_stale error | timeout | invalid_header | updating | http_500 | http_502 | http_503 | http_504 | http_403 | http_404 | http_429 | off ...;
Default: proxy_cache_use_stale off;
Context: http, server, location
```

缓存 demo 示例:
```conf
http {
    ....
    proxy_cache_path /data/home/user00/openresty/nginx/test_proxy_cache levels=1:2 use_temp_path=off keys_zone=test_cache_zone:1m inactive=1m max_size=1G;
    ....
    
    server {
        location / {
            proxy_cache   test_cache_zone;
            proxy_cache_valid any 10m;
            proxy_pass http://test_server_upstream;
        }
    }
}
```

# 四、location匹配规则
匹配uri的优先顺序，从高到底:
* 0, `location = /uri`, 精确匹配优先级最高
* 1, `location ^~ /uri`, 前缀匹配优先级次之, 前缀匹配内部按最大匹配长度来确认
* 2, `location ~ /uri`, 正则匹配, 正则匹配按配置文件的顺序, 一旦匹配到直接使用
* 2, `location ~* /uri`, 特殊的正则匹配, 不区分大小写, 和普通正则匹配同一优先级
* 3, `location /uri`, 优先级低于正则匹配的前缀匹配, 前缀匹配内部按最大匹配长度来确认
* 4, `location /`, 优先级最低的, 通常进行默认处理

# 五、http_map的使用

# 六、Nginx执行阶段
## 6.1 十一个处理阶段
* 1) `NGX_HTTP_POST_READ_PHASE`, 读取到完整的请求头后进行处理的阶段
* 2) `NGX_HTTP_SERVER_REWRITE_PHASE`,  还未进行URI的Location匹配前的阶段，可以进行URI重写以进行Location匹配
* 3) `NGX_HTTP_FIND_COFIG_PHASE`, 根据URI匹配Location的阶段(不可挂载)
* 4) `NGX_HTTP_REWRITE_PHASE`, URI和Location匹配后的重写阶段
* 5) `NGX_HTTP_POST_REWRITE_PHASE`, 重写提交阶段, 若4)进行了重写，通过5)将会回到3), 重新进行匹配。当循环匹配了10次, Nginx认为死循环，返回状态码500(不可挂载)
* 6) `NGX_HTTP_PREACCESS_PHASE`, 处理访问权限前, 让http模块可以介入处理, 该阶段本人没有实际接触过
* 7) `NGX_HTTP_ACCESS_PHASE`, 该阶段用于判断请求的访问权限, 很多网关权限校验都做在这一层
* 8) `NGX_HTTP_POST_ACCESS_PHASE`, 若7)不允许访问，8)将会构造拒绝访问的回包(不可挂载)
* 9) `NGX_HTTP_TRY_FILES_PHASE`, 为try-files配置项设立的, 用于处理静态文件(不可挂载)
* 10) `NGX_HTTP_CONTENT_PHASE`, 处理HTTP请求内容的阶段, 可以在返回client前控制具体内容, 例如gzip压缩内容就在该阶段
* 11) `NGX_HTTP_LOG_PHASE`, 处理请求完成后的日志阶段, 比如ngx_http_log_module的handler在该阶段记录`access_log`

## 6.2 常用场景
### 1).*响应信息收集*
对于响应数据或许需要进行收集，例如自定义日志等。需要放在`NGX_HTTP_LOG_PHASE`进行处理，而不能放到`NGX_HTTP_CONTENT_PHASE`：
* 因为在上一个阶段，已经处理了回包，该阶段不会影响请求的耗时
* 在该阶段，我们可以获取到该次请求的所有信息(包括请求信息以及回包信息)

# 七、日志记录

# 八、常用模块

# 九、一些常见的问题
## 9.1.proxy_pass指令
### 1).proxy_pass的uri
proxy_pass的uri写法一般有2种:
```conf
location /location-uri {
    # 配置1: 忽略uri
    proxy_pass http://server;

    # 配置2: 写明uri
    proxy_pass http://server/pass-uri;
}
```
这2者有较大的区别:
* 配置1, 忽略uri的情况下，会使用request自身的uri
* 配置2, 写明uri的情况下(包括一个/)，会从request自身的uri中剔除掉匹配部分的uri, 再拼接到proxy_pass的后面。因此如果使用了正则方式匹配uri，则该配置禁止使用(ngx加载配置报错)。

### 2).cookie丢失问题
有一次在使用proxy_pass时，发生了RS没有拿到对应cookie的问题，但是从chorme-dev-tools中又观察到cookie实际上是发送出去了的。这有可能是nginx关掉了对于headers的转发，检查配置文件后，发行在server块中配置了`proxy_pass_request_headers off;`, 这将会导致header不会被转发到代理服务器，进而丢失cookie。该配置项，默认是`on`。

### 3).`uri`重定向`uri/`的问题

## 9.2 set指令
### 1).if语句块的生效顺序问题

## 9.3 upstream语句块
### 1).*主动探测节点存活*
### 2).*被动检测节点存活*

## 9.4 rewrite指令
rewrite可以在server/location/if语句块中使用。
* 指令格式:
    * `rewrite regex replacement [flag];`, 当请求的uri和`regex`匹配时，uri将会被改写为`replacement`。

### 1).flag
rewrite可以支持4中flag:
* last(default), 将uri改变为replacement，并在location中重新到进行rewrite。
* break, 将uri改变为replacement，并在location中重新到进行rewrite，若该location已经进行过一次rewrite则该location不再进行rewrite。
* redirect, 产生一个302重定向的响应，让浏览器跳到replacement。
* permanent, 产生一个301重定向的响应，让浏览器跳到replacement。301为永久重定向，一般是为了SEO。
