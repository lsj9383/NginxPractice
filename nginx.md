# 一、Nginx概述

# 二、内置变量
下面只列出的本人使用过的内置Nginx变量，其他内置变量并未列出，将会根据本人情况持续更新。更全面的内置变量可以看[官方文档](http://nginx.org/en/docs/varindex.html)。

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
下面会列出每个语句块中的常用配置。

## 1.全局配置
## 2.events块配置
## 3.http块配置
## 4.server块配置
## 5.upstream块配置
## 6.location块配置
## 7.if语句块

## 7.配置Demo
### 1).*目录树*
### 2).*各文件配置*

# 四、不常用配置

# 五、location匹配规则
匹配uri的优先顺序，从高到底:
* 0, `location = /uri`, 精确匹配优先级最高
* 1, `location ^~ /uri`, 前缀匹配优先级次之, 前缀匹配内部按最大匹配长度来确认
* 2, `location ~ /uri`, 正则匹配, 正则匹配按配置文件的顺序, 一旦匹配到直接使用
* 2, `location ~* /uri`, 特殊的正则匹配, 不区分大小写, 和普通正则匹配同一优先级
* 3, `location /uri`, 优先级低于正则匹配的前缀匹配, 前缀匹配内部按最大匹配长度来确认
* 4, `location /`, 优先级最低的, 通常进行默认处理

# 六、http_map的使用

# 七、Nginx执行阶段
## 1.十一个处理阶段
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

## 3.常用场景
### 1).*响应信息收集*
对于响应数据或许需要进行收集，例如自定义日志等。需要放在`NGX_HTTP_LOG_PHASE`进行处理，而不能放到`NGX_HTTP_CONTENT_PHASE`：
* 因为在上一个阶段，已经处理了回包，该阶段不会影响请求的耗时
* 在该阶段，我们可以获取到该次请求的所有信息(包括请求信息以及回包信息)

# 八、日志记录

# 九、常用模块

# 九、proxy_pass的一些坑
## 1.是否重写uri的问题

## 2.cookie丢失问题

## 3.uri被重定向到斜杠后缀的问题

# 十、upstream的节点在线检查

# 十一、set指令的一些问题
