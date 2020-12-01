# HTTP 框架的执行流程s

<!-- TOC -->

- [HTTP 框架的执行流程s](#http-框架的执行流程s)
    - [新建连接](#新建连接)
    - [第一次可读事件的处理](#第一次可读事件的处理)
    - [接收 HTTP 请求行](#接收-http-请求行)
    - [接收 HTTP 头部](#接收-http-头部)
    - [处理 HTTP 请求](#处理-http-请求)
    - [subrequest 和 post 请求](#subrequest-和-post-请求)
    - [处理 HTTP 包体](#处理-http-包体)
    - [发送 HTTP 响应](#发送-http-响应)
    - [结束 HTTP 请求](#结束-http-请求)

<!-- /TOC -->


## 新建连接

用户发起 TCP 链接请求时，事件框架会负责把 TCP 连接建立起来，并回调监听方法。对于 HTTP 模块建立的连接，回调方法是 `ngx_http_init_connection` 方法：

```c
void ngx_http_init_connection(ngx_connection_t *c);
```

在该函数中，会设置连接对应的读方法和定时器：

1. 将新建立的连接 c 的读事件设置为 ngx_http_init_request，以及写事件为 ngx_http_empty_handler。
   - `c->read->handler = ngx_http_init_request`。
   - `c->write->handler = ngx_http_empty_handler`，由于此时处于接收 HTTP 头部，所以不应该处理任何写事件，该 handler 仅仅只会记录日志。
1. 如果读事件的 ready 为 1，代表这个套接字已经存在发过来的数据，因此可以调用 `ngx_http_init_request` 处理数据。
1. 设置读事件的定时器，超时时间为 `client_header_timeout`。
1. 将新连接的读事件添加到 EPOLL 中。

## 第一次可读事件的处理

建立连接后，第一次出现可读事件，会触发回调方法 `ngx_http_init_request`。

**注意：**

- 在 `ngx_http_init_connection` 中处理新建立连接事件时，由于TCP三次握手结束时，本身就能在最后的步骤中 client 携带数据到 server，因此在该建立连接的回调中也会调用 `ngx_http_init_request`，来处理建立连接时的数据。

```c
static void ngx_http_init_request(ngx_event_t *reg);
```

该函数的主要目的是为了构造请求对象、分配相关内存、创造上下文结构体的存放空间，并开始准备获取请求行。

1. 如果由于读事件超时（ngx_event_t->timeout == 1 ?），导致触发了 ngx_http_init_request 的执行，直接调用 `ngx_http_close_request` 进行关闭请求。
1. 如果没有超时，进行正常的处理，构建 ngx_http_request_t 对结构体，并将设置 `connection->data = ngx_http_request_t`。
1. 获取 default server 的 ngx_http_core_module 的 srv 配置。
1. 将读事件修改为 `ngx_http_process_request_line` 处理请求行。
1. 为连接创建用户态的内存空间，用于将内核缓冲区的 TCP 流复制到用户态的内存中。这是在 connection 中分配用户态内存空间，该内存大小与配置指令 `client_header_buffer_size` 指定的大小一致。ngx_connection_t->buffer 和 ngx_http_request_t->header_in 共同指向该内存。
1. 为 ngx_http_request_t 构建内存池，这个内存池用于存放和请求相关的数据。
1. 初始化请求上下文数组，用于支持所有模块配置请求上下文。
1. 调用 `ngx_http_process_request_line` 主动处理请求行。

**注意：**

- 现在的 Nginx 源码已经没有 `ngx_http_init_request` 了，取而代之的是 `ngx_http_wait_request_handler`。
- 现在的 Nginx 源码是在 `ngx_http_init_connection` 中获取的 default server 的 ngx_http_core_module 的 srv 配置。
- 书上的流程和现在的 Nginx 代码出入有点大。

## 接收 HTTP 请求行

在接收到 HTTP 数据时，该数据必然是请求行，会触发 `ngx_http_process_request_line` 回调。该回调函数的目的在于解析请求行的信息，对于不同的 HTTP 版本可能会有不同的处理，并且在解析完请求行的同时，会初始化用于解析 HTTP 请求头的相关结构体。

请求行格式：

```http
GET /uri HTTP/1.1
```

**注意**

- 在 `ngx_http_init_request` 接收到可读事件的时候，通常可能已经携带数据了，所以在可读事件回调中会主动触发一次 `ngx_http_process_request_line`。
- 请求行的长度是不固定的，与 URI 长度相关，内核套接字缓冲区大小未必可以一次接收完所有的 HTTP 请求行，因此可能会触发多次 `ngx_http_process_request_line` 来解析请求行。

`ngx_http_process_request_line` 处理步骤如下：

1. 检查读事件是否超时，超时时间为 `client_header_timeout`，若超时，则调用 `ngx_http_close_request` 结束请求。
1. r->header_in 接收缓冲区中若为空，则通过 recv 方法将 Linux Socket 缓冲区中的数据读取 header_in 缓冲区中。
1. recv 如果知道连接是否出错或者客户端已经关闭了连接，若存在一场则调用则结束。
1. recv 如果未能读出数据，则会将读事件添加到定时器和 epoll 中（每次触发事件时，都会从定时器或epoll中移除，因此每次要重新触发，都需要先添加）。
1. recv 如果接收到数据，则会调用 `ngx_http_parse_request_line` 解析请求行，并根据该函数的返回判断是否请求行接收完毕，或者请求行接收失败，或者未接收完毕，需要进一步接收。
1. 如果需要进一步接收请求行，判断 r->header_in 缓冲区是否耗尽，如果耗尽则调用 `ngx_http_alloc_large_header_buffer` 分配更大的接收缓冲区。配置 `large_client_header_buffers` 用来指定该大缓冲区的大小（对于大型请求的客户端才会启用，一般情况下 `client_header_buffers` 就够用了）。
1. 解析完成数据后，将解析处理出的 uri、method_name、http_proto 等等参数设置到 ngx_http_request_t 中。
1. 对于小于 HTTP 1.0 的请求，不会解析 HTTP 请求头，这里省略对于小于 HTTP 1.0 请求的处理。
1. 对于 HTTP 1.0 及其后续的请求，会初始化用于接收请求头的相关结构。
1. 设置回调方法为 `ngx_http_process_request_headers`，开始接收 HTTP 头的数据。
1. 主动调用 `ngx_http_process_request_headers` 处理 HTTP 头的数据（因为处理请求行的时候，缓冲区中可能已经有了 HTTP 请求头的数据）。

## 接收 HTTP 头部

通过 `ngx_http_process_request_headers` 回调函数处理 HTTP 头部。该回调的目的主要是解析 HTTP 头部，根据 Host 找到对应 server 的 srv 级别配置，检查 HTTP 头的合法性等。

这是一个完整的 HTTP 请求：

```http
GET /uri HTTP/1.1
cread: xxxx
username: ttt
content-length: 4

test
```

很明显，HTTP 头部也是一个可变长的数据，请求行和 body 之间都是通过换行来区分的。

请求头的大小是有限制，首次缓冲区分配 `client_buffer_size`，如果不够，会再分配 `large_client_buffer_size` 的缓冲区，如果请求头超过了 `large_client_buffer_size` 则 Nginx 将会返回错误。

`ngx_http_process_request_headers` 的相关处理步骤：

1. 判断是否读事件超时，超时时间为 `client_header_timeout`。如果超时，关闭请求。
1. 检查 r->header_in 缓冲区是否耗尽，如果耗尽则调用 `ngx_http_alloc_large_header_buffer` 分配更大更多的缓冲区。如果无法分配更大的缓冲区，则关闭请求。
1. 通过 recv 接收 Socket 内核缓冲区中的数据，复制到 r->header_in 中。
1. 如果 recv 发现连接错误或者关闭，则关闭请求。
1. 如果 recv 发现连接没有数据，需要进一步接收数据，则将事件重新注册到 epoll 和定时器中。
1. 如果 recv 读出数据，则通过 `ngx_http_parse_header_line` 解析 HTTP 请求头。
1. 当解析出 Header 中的 Host 字段时，会去找到对应的虚拟主机（即对应 server {} 的 ngx_http_core_srv_conf_t 结构体）。
1. 检查 HTTP Headers 的合法性。
1. 调用 `ngx_http_process_request` 开始使用 HTTP 各个模块处理请求。

**注意：**

- 请求头解析完毕后，读写事件的设置在 `ngx_http_process_request` 函数中完成。

## 处理 HTTP 请求

## subrequest 和 post 请求

## 处理 HTTP 包体

## 发送 HTTP 响应

## 结束 HTTP 请求
